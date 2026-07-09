# Movie Ticket Booking System

This is the concurrency showcase problem in this knowledge base. The interesting part isn't the entity model (movies/shows/seats are straightforward) — it's that two users can click "book" on the same seat within milliseconds of each other, and the design must make double-booking *structurally impossible*, not just unlikely. See `../06-concurrency-essentials.md` for the underlying lock primitives; this file is where you apply them to a real race.

## Requirements

- "Single theater or multi-theater/multi-city?" → **You decide**: model one `Show` (movie + screen + start time) as the booking unit; multi-theater is a pure scaling follow-up (see below), not a core design change.
- "What happens between 'select seats' and 'payment confirmed'?" → **You decide**: seats enter a temporary `LOCKED` state for a bounded hold window (e.g., 5 minutes); payment success transitions to `BOOKED`, payment failure or timeout releases back to `AVAILABLE`. This hold mechanic is the crux of the problem.
- "Can a seat be locked by more than one user at once?" → **You decide**: no — lock acquisition is exclusive and atomic per seat; this is the concurrency requirement the whole design centers on.
- "Seat pricing?" → **You decide**: price varies by seat type (regular/premium/recliner), pluggable per show (a recliner in a weekday matinee vs a weekend premiere may price differently).
- In scope: browsing shows, selecting seats, locking with TTL, payment confirmation/failure, auto-release on timeout, seat-type pricing, notifying on booking confirmation.
- Out of scope: actual payment gateway integration (assume a `PaymentGateway` interface that returns success/failure), seat recommendation, discounts/coupons, refunds/cancellation flow.

## Core entities & relationships

```
Theater
  └─ has-a[*] Screen

Screen
  ├─ has-a[*] Seat
  └─ has-a[*] Show

Show
  ├─ has-a[1] Movie
  ├─ has-a[1] Screen
  └─ has-a[1] startTime

Seat
  ├─ has-a[1] SeatType (enum: REGULAR, PREMIUM, RECLINER)
  ├─ has-a[1] SeatState (enum: AVAILABLE, LOCKED, BOOKED)   -- explicit state machine, not booleans
  └─ has-a[1] Lock (ReentrantLock / threading.Lock)          -- one lock per seat, guards state transitions

SeatLockManager
  ├─ locks[*] Seat
  └─ schedules[*] auto-release timers (one per lock, keyed by seat+bookingId)

PricingStrategy (interface)
  └─ implemented-by[*] one per SeatType (or per show)

Booking
  ├─ has-a[*] Seat
  ├─ has-a[1] User
  └─ has-a[1] BookingStatus (PENDING, CONFIRMED, FAILED)

BookingService
  ├─ uses-a[1] SeatLockManager
  ├─ uses-a[1] PaymentGateway (interface)
  └─ notifies[*] BookingObserver
```

`Seat` owns its own `Lock` rather than the system using one global lock — this is the deliberate concurrency design choice: booking seat A3 and booking seat C7 for two different users must not block each other, only contention on the *same* seat should serialize. State is modeled as an explicit `SeatState` enum with a guarded transition method, not a `boolean isLocked` + `boolean isBooked` pair — booleans allow invalid combinations (`isLocked=true, isBooked=true`) that an enum with one authoritative value cannot.

## Design patterns applied

- [State](../patterns/03-behavioral-patterns.md#state) — `Seat` transitions `AVAILABLE → LOCKED → BOOKED` (or `LOCKED → AVAILABLE` on timeout/failure); modeling this as an explicit state machine with guarded transitions, rather than boolean flags, is exactly what prevents the class of bug where a seat is simultaneously "locked" and "available" due to a missed flag reset.
- [Strategy](../patterns/03-behavioral-patterns.md#strategy) — `PricingStrategy` varies price by `SeatType` (and could vary further by show/time-of-day) independent of the booking flow; a new seat tier or dynamic/surge pricing rule is a new class, not a new `if` branch in `BookingService`.
- [Observer](../patterns/03-behavioral-patterns.md#observer) — `BookingObserver` implementations (email/SMS/push) get notified on booking confirmation without `BookingService` knowing or caring how notification happens; also usable to push live seat-map updates to other clients viewing the same show.

## Java implementation

```java
public enum SeatType { REGULAR, PREMIUM, RECLINER }
public enum SeatState { AVAILABLE, LOCKED, BOOKED }
public enum BookingStatus { PENDING, CONFIRMED, FAILED }
```

```java
import java.util.concurrent.locks.ReentrantLock;

/** Owns its own lock — contention on seat A3 never blocks a booking for seat C7. */
public class Seat {
    private final String id;
    private final SeatType type;
    private volatile SeatState state = SeatState.AVAILABLE;
    private volatile String lockedByBookingId; // which booking currently holds the lock, for release validation
    private final ReentrantLock lock = new ReentrantLock();

    public Seat(String id, SeatType type) { this.id = id; this.type = type; }

    public String getId() { return id; }
    public SeatType getType() { return type; }
    public SeatState getState() { return state; }

    /** Atomic check-and-transition AVAILABLE -> LOCKED. Returns false if already taken. */
    public boolean tryLock(String bookingId) {
        lock.lock();
        try {
            if (state != SeatState.AVAILABLE) return false; // already LOCKED or BOOKED by someone else
            state = SeatState.LOCKED;
            lockedByBookingId = bookingId;
            return true;
        } finally {
            lock.unlock();
        }
    }

    /** Called on payment success. Only the booking that holds the lock may confirm it. */
    public boolean confirm(String bookingId) {
        lock.lock();
        try {
            if (state != SeatState.LOCKED || !bookingId.equals(lockedByBookingId)) return false;
            state = SeatState.BOOKED;
            return true;
        } finally {
            lock.unlock();
        }
    }

    /** Called on payment failure or TTL timeout. No-op if already booked or re-locked by someone else. */
    public void release(String bookingId) {
        lock.lock();
        try {
            if (state == SeatState.LOCKED && bookingId.equals(lockedByBookingId)) {
                state = SeatState.AVAILABLE;
                lockedByBookingId = null;
            }
        } finally {
            lock.unlock();
        }
    }
}
```

```java
import java.util.*;
import java.util.concurrent.*;

/** Coordinates seat locking with a TTL-based auto-release. */
public class SeatLockManager {
    private static final long HOLD_MILLIS = 5 * 60 * 1000;
    private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);

    /**
     * Locks all-or-nothing: if any seat is unavailable, roll back the ones already
     * acquired so a partial hold never blocks other users on seats this booking won't get.
     */
    public boolean lockSeats(List<Seat> seats, String bookingId) {
        List<Seat> acquired = new ArrayList<>();
        for (Seat seat : seats) {
            if (seat.tryLock(bookingId)) {
                acquired.add(seat);
            } else {
                for (Seat toRelease : acquired) toRelease.release(bookingId);
                return false;
            }
        }
        for (Seat seat : seats) {
            scheduler.schedule(() -> seat.release(bookingId), HOLD_MILLIS, TimeUnit.MILLISECONDS);
        }
        return true;
    }

    public void confirmSeats(List<Seat> seats, String bookingId) {
        for (Seat seat : seats) seat.confirm(bookingId); // scheduled release becomes a no-op post-confirm
    }

    public void releaseSeats(List<Seat> seats, String bookingId) {
        for (Seat seat : seats) seat.release(bookingId);
    }
}
```

```java
public interface PricingStrategy {
    java.math.BigDecimal priceFor(Seat seat, Show show);
}

public class SeatTypePricingStrategy implements PricingStrategy {
    private final Map<SeatType, java.math.BigDecimal> basePrices;
    public SeatTypePricingStrategy(Map<SeatType, java.math.BigDecimal> basePrices) {
        this.basePrices = basePrices;
    }
    @Override
    public java.math.BigDecimal priceFor(Seat seat, Show show) {
        return basePrices.get(seat.getType());
    }
}

public interface PaymentGateway {
    boolean charge(String userId, java.math.BigDecimal amount);
}

public interface BookingObserver {
    void onBookingConfirmed(Booking booking);
}

public class Booking {
    private final String id;
    private final String userId;
    private final List<Seat> seats;
    private volatile BookingStatus status = BookingStatus.PENDING;

    public Booking(String id, String userId, List<Seat> seats) {
        this.id = id; this.userId = userId; this.seats = seats;
    }
    public String getId() { return id; }
    public BookingStatus getStatus() { return status; }
    void setStatus(BookingStatus s) { status = s; }
}

public class BookingService {
    private final SeatLockManager lockManager;
    private final PricingStrategy pricingStrategy;
    private final PaymentGateway paymentGateway;
    private final List<BookingObserver> observers = new ArrayList<>();

    public BookingService(SeatLockManager lockManager, PricingStrategy pricingStrategy, PaymentGateway paymentGateway) {
        this.lockManager = lockManager;
        this.pricingStrategy = pricingStrategy;
        this.paymentGateway = paymentGateway;
    }

    public void addObserver(BookingObserver observer) { observers.add(observer); }

    /** The end-to-end flow: lock -> charge -> confirm-or-release. */
    public Booking book(String userId, Show show, List<Seat> seats) {
        String bookingId = UUID.randomUUID().toString();
        Booking booking = new Booking(bookingId, userId, seats);

        if (!lockManager.lockSeats(seats, bookingId)) {
            booking.setStatus(BookingStatus.FAILED); // someone else holds >= 1 requested seat
            return booking;
        }

        java.math.BigDecimal total = seats.stream()
                .map(s -> pricingStrategy.priceFor(s, show))
                .reduce(java.math.BigDecimal.ZERO, java.math.BigDecimal::add);

        if (paymentGateway.charge(userId, total)) {
            lockManager.confirmSeats(seats, bookingId);
            booking.setStatus(BookingStatus.CONFIRMED);
            for (BookingObserver obs : observers) obs.onBookingConfirmed(booking);
        } else {
            lockManager.releaseSeats(seats, bookingId); // payment failed -> release the hold immediately
            booking.setStatus(BookingStatus.FAILED);
        }
        return booking;
    }
}
```

## Python implementation

```python
from __future__ import annotations
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from decimal import Decimal
from enum import Enum, auto
import threading
import time
import uuid


class SeatType(Enum):
    REGULAR = auto()
    PREMIUM = auto()
    RECLINER = auto()


class SeatState(Enum):
    AVAILABLE = auto()
    LOCKED = auto()
    BOOKED = auto()


class BookingStatus(Enum):
    PENDING = auto()
    CONFIRMED = auto()
    FAILED = auto()


class Seat:
    """Owns its own threading.Lock — contention on one seat never blocks another."""
    def __init__(self, seat_id: str, seat_type: SeatType):
        self.id = seat_id
        self.type = seat_type
        self.state = SeatState.AVAILABLE
        self._locked_by: str | None = None
        self._lock = threading.Lock()

    def try_lock(self, booking_id: str) -> bool:
        with self._lock:
            if self.state != SeatState.AVAILABLE:
                return False
            self.state = SeatState.LOCKED
            self._locked_by = booking_id
            return True

    def confirm(self, booking_id: str) -> bool:
        with self._lock:
            if self.state != SeatState.LOCKED or self._locked_by != booking_id:
                return False
            self.state = SeatState.BOOKED
            return True

    def release(self, booking_id: str) -> None:
        with self._lock:
            if self.state == SeatState.LOCKED and self._locked_by == booking_id:
                self.state = SeatState.AVAILABLE
                self._locked_by = None


class SeatLockManager:
    """TTL-based auto-release via a background timer thread per lock (no ScheduledExecutorService
    equivalent in the stdlib; threading.Timer is the direct analog for a single-process design)."""
    HOLD_SECONDS = 5 * 60

    def lock_seats(self, seats: list[Seat], booking_id: str) -> bool:
        acquired: list[Seat] = []
        for seat in seats:
            if seat.try_lock(booking_id):
                acquired.append(seat)
            else:
                for taken in acquired:
                    taken.release(booking_id)  # all-or-nothing: roll back partial hold
                return False
        for seat in seats:
            timer = threading.Timer(self.HOLD_SECONDS, seat.release, args=(booking_id,))
            timer.daemon = True
            timer.start()
        return True

    def confirm_seats(self, seats: list[Seat], booking_id: str) -> None:
        for seat in seats:
            seat.confirm(booking_id)  # pending release timer becomes a no-op post-confirm

    def release_seats(self, seats: list[Seat], booking_id: str) -> None:
        for seat in seats:
            seat.release(booking_id)


class PricingStrategy(ABC):
    @abstractmethod
    def price_for(self, seat: Seat, show: "Show") -> Decimal: ...


class SeatTypePricingStrategy(PricingStrategy):
    def __init__(self, base_prices: dict[SeatType, Decimal]):
        self.base_prices = base_prices

    def price_for(self, seat: Seat, show: "Show") -> Decimal:
        return self.base_prices[seat.type]


class PaymentGateway(ABC):
    @abstractmethod
    def charge(self, user_id: str, amount: Decimal) -> bool: ...


class BookingObserver(ABC):
    @abstractmethod
    def on_booking_confirmed(self, booking: "Booking") -> None: ...


@dataclass
class Show:
    id: str
    movie_title: str
    start_time: float


@dataclass
class Booking:
    id: str
    user_id: str
    seats: list[Seat]
    status: BookingStatus = BookingStatus.PENDING


class BookingService:
    def __init__(self, lock_manager: SeatLockManager, pricing: PricingStrategy, gateway: PaymentGateway):
        self.lock_manager = lock_manager
        self.pricing = pricing
        self.gateway = gateway
        self._observers: list[BookingObserver] = []

    def add_observer(self, observer: BookingObserver) -> None:
        self._observers.append(observer)

    def book(self, user_id: str, show: Show, seats: list[Seat]) -> Booking:
        booking_id = str(uuid.uuid4())
        booking = Booking(id=booking_id, user_id=user_id, seats=seats)

        if not self.lock_manager.lock_seats(seats, booking_id):
            booking.status = BookingStatus.FAILED  # someone else holds >= 1 requested seat
            return booking

        total = sum((self.pricing.price_for(s, show) for s in seats), Decimal(0))

        if self.gateway.charge(user_id, total):
            self.lock_manager.confirm_seats(seats, booking_id)
            booking.status = BookingStatus.CONFIRMED
            for obs in self._observers:
                obs.on_booking_confirmed(booking)
        else:
            self.lock_manager.release_seats(seats, booking_id)  # payment failed -> release immediately
            booking.status = BookingStatus.FAILED
        return booking
```

## Sample walkthrough

```python
seat_a1 = Seat("A1", SeatType.PREMIUM)
seat_a2 = Seat("A2", SeatType.PREMIUM)
show = Show(id="s1", movie_title="Dune: Part Three", start_time=time.time() + 3600)

class AlwaysApprove(PaymentGateway):
    def charge(self, user_id, amount): return True

class ConsoleNotifier(BookingObserver):
    def on_booking_confirmed(self, booking):
        print(f"Booking {booking.id} confirmed for {booking.user_id}")

service = BookingService(
    SeatLockManager(),
    SeatTypePricingStrategy({SeatType.PREMIUM: Decimal("15.00"), SeatType.REGULAR: Decimal("10.00")}),
    AlwaysApprove(),
)
service.add_observer(ConsoleNotifier())

booking = service.book(user_id="alice", show=show, seats=[seat_a1, seat_a2])
assert booking.status == BookingStatus.CONFIRMED
assert seat_a1.state == SeatState.BOOKED and seat_a2.state == SeatState.BOOKED

# A second user tries the same seats after alice already locked them -> fails atomically
second_booking = service.book(user_id="bob", show=show, seats=[seat_a1, seat_a2])
assert second_booking.status == BookingStatus.FAILED
```

## Follow-up questions

- **"Two users click 'book' on the same seat at the exact same instant — walk through what actually happens."** Both threads call `seat.try_lock(bookingId)`; `ReentrantLock`/`threading.Lock` in `try_lock` serializes them so only one thread's check-`state==AVAILABLE`-then-set-`LOCKED` executes as one atomic unit — the second thread's `try_lock` runs *after* the first has already flipped `state` to `LOCKED`, sees `state != AVAILABLE`, and returns `false`. The lock is what turns a check-then-act race into a single atomic operation; without it, both threads could pass the `state == AVAILABLE` check before either writes `LOCKED`.
- **"Payment fails after the seat was locked — what happens to the seat?"** `BookingService.book` catches the `false` return from `paymentGateway.charge` and calls `lockManager.releaseSeats`, which calls `seat.release(bookingId)` — guarded by `bookingId` match so a stale/late release call can't accidentally free a seat some *other* booking has since re-locked.
- **"User closes the tab / never completes payment — seat held forever?"** The TTL auto-release (`ScheduledExecutorService.schedule` in Java, `threading.Timer` in Python) fires `release(bookingId)` after `HOLD_MILLIS` regardless of what the user does; `release` is a no-op if the seat was already confirmed, so a booking that completes just before the timer fires is unaffected.
- **"Extend to multiple screens/theaters, and to searching shows by city."** `Theater has-a[*] Screen has-a[*] Show` already scales horizontally — add a `TheaterRepository`/search index keyed by city + movie + date; no change to the locking or booking core, since locking is already scoped to individual `Seat` objects, not global state.
- **"What if the interviewer requires this to work across multiple app server instances (not just multi-threaded within one process)?"** In-process `Lock` objects don't coordinate across machines — the fix is a distributed lock (Redis `SETNX`/Redlock, or a DB row-level lock via `SELECT ... FOR UPDATE` on the seat row) with the same TTL-hold semantics; flag this explicitly as a scope question rather than silently assuming single-instance, since it changes the locking primitive entirely.

## Common mistakes on this problem

- Modeling seat availability as a `boolean isBooked` (or `isBooked` + `isLocked`) instead of an explicit `SeatState` enum — booleans permit invalid combined states and make the timeout-release transition error-prone to reason about.
- Checking `if (seat.getState() == AVAILABLE) { seat.setState(LOCKED); }` as two separate statements instead of one atomic `tryLock` — this is the check-then-act race that causes the exact double-booking bug the problem is designed to test for; the check and the mutation must happen under the same lock acquisition.
- Using one global lock over the entire seat map for booking — technically prevents double-booking but serializes unrelated bookings for different seats/shows, killing throughput; the fix is per-seat (or per-show) lock granularity.
- Forgetting the auto-release path entirely — a design that locks seats on selection but has no timeout/failure release leaves seats permanently unavailable after an abandoned checkout, which is the first thing an interviewer will probe ("what if the user just closes the browser?").

## Continue

Next: [08-library-management-system.md](08-library-management-system.md)
