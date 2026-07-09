# Parking Lot System

THE canonical LLD problem — mainly because it has just enough variability (vehicle types, pricing schemes, multi-floor layout, concurrency at the last-spot edge case) to justify 3-4 patterns without forcing any of them.

## Requirements

- "How many floors / spots per floor?" → **You decide**: assume a fixed number of floors known at construction time, each with a fixed spot layout. Dynamic floor add/remove is out of scope but the design shouldn't actively block it (floors live in a `List`, not baked into constants).
- "Which vehicle types?" → Motorcycle, Car, Truck, Electric (any vehicle, gets a charging-capable spot).
- "One spot size fits all, or does size matter?" → Spots are Small/Medium/Large; Motorcycle needs Small+, Car needs Medium+, Truck needs Large only, Electric needs Medium+ **and** a charger.
- "Pricing — flat hourly or something fancier?" → Support both flat-hourly and tiered (first N hours at rate X, beyond that rate Y) from day one, via a pluggable strategy — this is the headline extensibility axis for this problem.
- "Multiple entry/exit gates?" → **You decide**: yes, multiple gates issue tickets and settle payment independently; this is what makes the "two vehicles race for the last spot" concurrency follow-up real rather than academic.
- "Payment — cash, card, wallet?" → Out of scope for the core design; model a `PaymentMethod` interface and assume `pay()` succeeds/fails synchronously. Don't build a payment gateway integration.
- "Reservations ahead of time?" → Out of scope — first-come-first-served allocation only.

**In scope:** multi-floor spot inventory, vehicle-to-spot-size matching, ticket issue/close, pluggable pricing, a display board reacting to floor-full events, thread-safe spot allocation.

**Out of scope:** reservations, payment gateway internals, dynamic re-numbering of floors/spots, distributed multi-lot coordination (see the concurrency follow-up for where that line is).

## Core entities & relationships

```
ParkingLot
  ├─ has-a[*] ParkingFloor
  ├─ has-a[1] PricingStrategy (interface)
  └─ has-a[*] DisplayBoard (interface, observer)

ParkingFloor
  └─ has-a[*] ParkingSpot

ParkingSpot
  ├─ has-a[1] SpotSize (enum)
  └─ has-a[0..1] Vehicle

Vehicle
  └─ has-a[1] VehicleType (enum)

ParkingTicket
  ├─ has-a[1] Vehicle
  └─ has-a[1] ParkingSpot

EntryGate / ExitGate
  └─ uses-a[1] ParkingLot
```

`Vehicle` is one concrete class with a `VehicleType` enum field, **not** a `Motorcycle`/`Car`/`Truck` subclass hierarchy. There's no behavior that differs by type — only two data lookups (required spot size, whether it needs a charger) — and an enum-with-behavior (see [Java enums §7](../03-java-oop-essentials.md#7-enums-with-behavior--not-just-constants) / [Python enums §5](../04-python-oop-essentials.md#5-enums--with-behavior-like-java)) captures that without the ceremony of four near-empty subclasses. Reach for subclassing only when a type needs genuinely different *logic*, not just different *data* — that's the elevator problem's `Piece` hierarchy in [problems/04](04-tic-tac-toe-and-chess.md), by contrast.

`ParkingSpot` holds a `Vehicle` by composition-of-reference (0..1), not composition-of-lifetime — the spot doesn't own the vehicle's lifecycle, it just tracks current occupancy. `ParkingTicket` is the join entity that survives after the vehicle leaves (for receipts/audit), so it holds copies of the relevant IDs rather than live references.

## Design patterns applied

- [Strategy](../patterns/03-behavioral-patterns.md#strategy) — `PricingStrategy` isolates "how do we bill this stay" from everything else; swapping flat-hourly for tiered-hourly (or adding weekend rates later) touches zero other classes, which is exactly the variability the interviewer is probing when they ask "what if pricing changes."
- [Factory Method](../patterns/01-creational-patterns.md#factory-method) (loosely) — `VehicleFactory.create(type, plate)` centralizes the type→construction mapping so `EntryGate` never has a `switch` on vehicle type. Being honest about the nuance: this is a **static/simple factory**, not the GoF Factory Method (no subclassed creators overriding a factory method) — there's no varying *creation algorithm* here, just one construction path per type, so the heavier pattern would be over-engineering. Worth saying this distinction out loud in an interview.
- [Singleton](../patterns/01-creational-patterns.md#singleton) — **judgment call: do not enforce it.** The natural instinct is `ParkingLot.getInstance()` with a private constructor. Don't do it: a hard-enforced Singleton bakes global mutable state into the class, makes unit tests (two lots, one per test) impossible without ceremony, and violates [DIP](../02-solid-principles.md#d--dependency-inversion-principle) since every collaborator now depends on a concrete static accessor instead of an injected instance. Instead, construct exactly one `ParkingLot` at the composition root (`main`, or a DI container's singleton-scoped bean) and hand it to every gate via constructor injection — "only one instance *exists*" is a deployment fact, not something the class itself needs to police. If you truly need cross-process single-instance guarantees (e.g., two `ParkingLot` processes racing to manage the same physical lot), that's a distributed-coordination problem (leader election / DB row lock), and a language-level Singleton can't solve it anyway — say this if the interviewer pushes on it.
- [Observer](../patterns/03-behavioral-patterns.md#observer) — `ParkingFloor` notifies subscribed `DisplayBoard`s on occupancy changes (spot taken/freed, floor full/available) instead of every caller polling `floor.isFull()`; new subscriber types (a mobile app push, a gate barrier controller) attach without touching `ParkingFloor`.

## Java implementation

```java
enum SpotSize { SMALL, MEDIUM, LARGE }

enum VehicleType {
    MOTORCYCLE(SpotSize.SMALL, false),
    CAR(SpotSize.MEDIUM, false),
    ELECTRIC(SpotSize.MEDIUM, true),
    TRUCK(SpotSize.LARGE, false);

    private final SpotSize requiredSize;
    private final boolean needsCharger;

    VehicleType(SpotSize requiredSize, boolean needsCharger) {
        this.requiredSize = requiredSize;
        this.needsCharger = needsCharger;
    }
    SpotSize requiredSize() { return requiredSize; }
    boolean needsCharger() { return needsCharger; }
}

final class Vehicle {
    private final String licensePlate;
    private final VehicleType type;
    Vehicle(String licensePlate, VehicleType type) {
        this.licensePlate = licensePlate;
        this.type = type;
    }
    VehicleType getType() { return type; }
    String getLicensePlate() { return licensePlate; }
}

final class VehicleFactory {
    private VehicleFactory() {}
    static Vehicle create(VehicleType type, String plate) {
        return new Vehicle(plate, type); // one construction path per type today; would grow into a real Factory Method only if that changes
    }
}

final class ParkingSpot {
    private final String id;
    private final SpotSize size;
    private final boolean hasCharger;
    private Vehicle occupant; // null == free

    ParkingSpot(String id, SpotSize size, boolean hasCharger) {
        this.id = id; this.size = size; this.hasCharger = hasCharger;
    }

    boolean fits(Vehicle v) {
        return size.ordinal() >= v.getType().requiredSize().ordinal()
            && (!v.getType().needsCharger() || hasCharger);
    }
    synchronized boolean tryOccupy(Vehicle v) { // per-spot lock: see Follow-up on concurrency
        if (occupant != null) return false;
        occupant = v;
        return true;
    }
    synchronized void vacate() { occupant = null; }
    boolean isFree() { return occupant == null; }
    String getId() { return id; }
}

interface FloorObserver {
    void onFloorFull(String floorId);
    void onFloorAvailable(String floorId);
}

final class ParkingFloor {
    private final String id;
    private final List<ParkingSpot> spots;
    private final List<FloorObserver> observers = new ArrayList<>();

    ParkingFloor(String id, List<ParkingSpot> spots) { this.id = id; this.spots = spots; }

    void subscribe(FloorObserver o) { observers.add(o); }

    // Returns the allocated spot, or null if the floor has none matching. Synchronized at floor
    // granularity to serialize the "scan + claim" sequence — see the concurrency follow-up below.
    synchronized ParkingSpot allocate(Vehicle v) {
        boolean wasFull = spots.stream().noneMatch(ParkingSpot::isFree);
        for (ParkingSpot s : spots) {
            if (s.isFree() && s.fits(v) && s.tryOccupy(v)) {
                if (spots.stream().noneMatch(ParkingSpot::isFree)) {
                    observers.forEach(o -> o.onFloorFull(id));
                }
                return s;
            }
        }
        return null;
    }

    synchronized void release(ParkingSpot s) {
        boolean wasFull = spots.stream().noneMatch(ParkingSpot::isFree);
        s.vacate();
        if (wasFull) observers.forEach(o -> o.onFloorAvailable(id));
    }
}

final class DisplayBoard implements FloorObserver {
    public void onFloorFull(String floorId) { System.out.println("Floor " + floorId + ": FULL"); }
    public void onFloorAvailable(String floorId) { System.out.println("Floor " + floorId + ": spot available"); }
}

interface PricingStrategy {
    double calculateFee(VehicleType type, Duration parkedDuration);
}

final class HourlyFlatPricing implements PricingStrategy {
    private final Map<VehicleType, Double> ratePerHour;
    HourlyFlatPricing(Map<VehicleType, Double> ratePerHour) { this.ratePerHour = ratePerHour; }
    public double calculateFee(VehicleType type, Duration d) {
        double hours = Math.ceil(d.toMinutes() / 60.0);
        return hours * ratePerHour.getOrDefault(type, 0.0);
    }
}

final class TieredHourlyPricing implements PricingStrategy {
    private final double baseHours, baseRate, overageRate;
    TieredHourlyPricing(double baseHours, double baseRate, double overageRate) {
        this.baseHours = baseHours; this.baseRate = baseRate; this.overageRate = overageRate;
    }
    public double calculateFee(VehicleType type, Duration d) {
        double hours = Math.ceil(d.toMinutes() / 60.0);
        if (hours <= baseHours) return hours * baseRate;
        return baseHours * baseRate + (hours - baseHours) * overageRate;
    }
}

final class ParkingTicket {
    private final String id;
    private final Vehicle vehicle;
    private final ParkingSpot spot;
    private final LocalDateTime entryTime;
    private LocalDateTime exitTime;

    ParkingTicket(Vehicle vehicle, ParkingSpot spot) {
        this.id = UUID.randomUUID().toString();
        this.vehicle = vehicle; this.spot = spot; this.entryTime = LocalDateTime.now();
    }
    void close() { this.exitTime = LocalDateTime.now(); }
    Duration duration() { return Duration.between(entryTime, exitTime != null ? exitTime : LocalDateTime.now()); }
    ParkingSpot getSpot() { return spot; }
    Vehicle getVehicle() { return vehicle; }
}

final class ParkingLot { // constructed once at the composition root — see Singleton discussion above
    private final List<ParkingFloor> floors;
    private final PricingStrategy pricingStrategy;
    private final Map<String, ParkingTicket> activeTickets = new ConcurrentHashMap<>();

    ParkingLot(List<ParkingFloor> floors, PricingStrategy pricingStrategy) {
        this.floors = floors; this.pricingStrategy = pricingStrategy;
    }

    ParkingTicket issueTicket(Vehicle vehicle) {
        for (ParkingFloor floor : floors) {
            ParkingSpot spot = floor.allocate(vehicle);
            if (spot != null) {
                ParkingTicket ticket = new ParkingTicket(vehicle, spot);
                activeTickets.put(ticket.getVehicle().getLicensePlate(), ticket);
                return ticket;
            }
        }
        throw new IllegalStateException("Lot full for " + vehicle.getType());
    }

    double settle(ParkingTicket ticket, PaymentMethod payment) {
        ticket.close();
        double fee = pricingStrategy.calculateFee(ticket.getVehicle().getType(), ticket.duration());
        if (!payment.pay(fee)) throw new IllegalStateException("Payment failed");
        floors.forEach(f -> f.release(ticket.getSpot())); // spot belongs to exactly one floor; release is a no-op elsewhere
        activeTickets.remove(ticket.getVehicle().getLicensePlate());
        return fee;
    }
}

interface PaymentMethod { boolean pay(double amount); }
```

## Python implementation

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from enum import Enum
from threading import Lock
from typing import Optional
import math, uuid

class SpotSize(Enum):
    SMALL = 1
    MEDIUM = 2
    LARGE = 3

class VehicleType(Enum):
    MOTORCYCLE = (SpotSize.SMALL, False)
    CAR = (SpotSize.MEDIUM, False)
    ELECTRIC = (SpotSize.MEDIUM, True)
    TRUCK = (SpotSize.LARGE, False)

    def __init__(self, required_size: SpotSize, needs_charger: bool):
        self.required_size = required_size
        self.needs_charger = needs_charger

@dataclass(frozen=True)
class Vehicle:
    license_plate: str
    type: VehicleType

class ParkingSpot:
    def __init__(self, spot_id: str, size: SpotSize, has_charger: bool):
        self.id = spot_id
        self.size = size
        self.has_charger = has_charger
        self._occupant: Optional[Vehicle] = None
        self._lock = Lock()  # per-spot lock; see ../06-concurrency-essentials.md#threadinglock--the-direct-equivalent-of-javas-synchronizedreentrantlock

    def fits(self, vehicle: Vehicle) -> bool:
        size_ok = list(SpotSize).index(self.size) >= list(SpotSize).index(vehicle.type.required_size)
        return size_ok and (not vehicle.type.needs_charger or self.has_charger)

    def try_occupy(self, vehicle: Vehicle) -> bool:
        with self._lock:
            if self._occupant is not None:
                return False
            self._occupant = vehicle
            return True

    def vacate(self) -> None:
        with self._lock:
            self._occupant = None

    def is_free(self) -> bool:
        return self._occupant is None

class FloorObserver(ABC):
    @abstractmethod
    def on_floor_full(self, floor_id: str) -> None: ...
    @abstractmethod
    def on_floor_available(self, floor_id: str) -> None: ...

class ParkingFloor:
    def __init__(self, floor_id: str, spots: list[ParkingSpot]):
        self.id = floor_id
        self.spots = spots
        self._observers: list[FloorObserver] = []
        self._lock = Lock()  # serializes scan-then-claim at floor granularity

    def subscribe(self, observer: FloorObserver) -> None:
        self._observers.append(observer)

    def allocate(self, vehicle: Vehicle) -> Optional[ParkingSpot]:
        with self._lock:
            for spot in self.spots:
                if spot.is_free() and spot.fits(vehicle) and spot.try_occupy(vehicle):
                    if all(not s.is_free() for s in self.spots):
                        for o in self._observers:
                            o.on_floor_full(self.id)
                    return spot
            return None

    def release(self, spot: ParkingSpot) -> None:
        with self._lock:
            was_full = all(not s.is_free() for s in self.spots)
            spot.vacate()
            if was_full:
                for o in self._observers:
                    o.on_floor_available(self.id)

class DisplayBoard(FloorObserver):
    def on_floor_full(self, floor_id: str) -> None:
        print(f"Floor {floor_id}: FULL")
    def on_floor_available(self, floor_id: str) -> None:
        print(f"Floor {floor_id}: spot available")

class PricingStrategy(ABC):
    @abstractmethod
    def calculate_fee(self, vehicle_type: VehicleType, parked_duration: timedelta) -> float: ...

class HourlyFlatPricing(PricingStrategy):
    def __init__(self, rate_per_hour: dict[VehicleType, float]):
        self.rate_per_hour = rate_per_hour
    def calculate_fee(self, vehicle_type, parked_duration):
        hours = math.ceil(parked_duration.total_seconds() / 3600)
        return hours * self.rate_per_hour.get(vehicle_type, 0.0)

class TieredHourlyPricing(PricingStrategy):
    def __init__(self, base_hours: float, base_rate: float, overage_rate: float):
        self.base_hours, self.base_rate, self.overage_rate = base_hours, base_rate, overage_rate
    def calculate_fee(self, vehicle_type, parked_duration):
        hours = math.ceil(parked_duration.total_seconds() / 3600)
        if hours <= self.base_hours:
            return hours * self.base_rate
        return self.base_hours * self.base_rate + (hours - self.base_hours) * self.overage_rate

@dataclass
class ParkingTicket:
    vehicle: Vehicle
    spot: ParkingSpot
    entry_time: datetime = field(default_factory=datetime.now)
    exit_time: Optional[datetime] = None
    id: str = field(default_factory=lambda: str(uuid.uuid4()))

    def close(self) -> None:
        self.exit_time = datetime.now()

    def duration(self) -> timedelta:
        return (self.exit_time or datetime.now()) - self.entry_time

class PaymentMethod(ABC):
    @abstractmethod
    def pay(self, amount: float) -> bool: ...

class ParkingLot:  # instantiated once at the composition root — no enforced Singleton
    def __init__(self, floors: list[ParkingFloor], pricing_strategy: PricingStrategy):
        self.floors = floors
        self.pricing_strategy = pricing_strategy
        self._active_tickets: dict[str, ParkingTicket] = {}

    def issue_ticket(self, vehicle: Vehicle) -> ParkingTicket:
        for floor in self.floors:
            spot = floor.allocate(vehicle)
            if spot:
                ticket = ParkingTicket(vehicle=vehicle, spot=spot)
                self._active_tickets[vehicle.license_plate] = ticket
                return ticket
        raise RuntimeError(f"Lot full for {vehicle.type}")

    def settle(self, ticket: ParkingTicket, payment: PaymentMethod) -> float:
        ticket.close()
        fee = self.pricing_strategy.calculate_fee(ticket.vehicle.type, ticket.duration())
        if not payment.pay(fee):
            raise RuntimeError("Payment failed")
        for floor in self.floors:
            if ticket.spot in floor.spots:
                floor.release(ticket.spot)
                break
        del self._active_tickets[ticket.vehicle.license_plate]
        return fee
```

## Sample walkthrough

```python
floor1 = ParkingFloor("F1", [ParkingSpot("F1-S1", SpotSize.SMALL, False),
                              ParkingSpot("F1-M1", SpotSize.MEDIUM, False),
                              ParkingSpot("F1-M2", SpotSize.MEDIUM, True)])
floor1.subscribe(DisplayBoard())

lot = ParkingLot([floor1], HourlyFlatPricing({VehicleType.CAR: 20.0, VehicleType.ELECTRIC: 25.0}))

car = Vehicle("KA-01-1234", VehicleType.CAR)
ticket = lot.issue_ticket(car)           # allocated to F1-M1 (first fitting free spot)
# ... two hours later ...
fee = lot.settle(ticket, CashPayment())  # HourlyFlatPricing.calculate_fee -> 2 * 20.0 = 40.0
```

## Follow-up questions

- **"Two vehicles arrive simultaneously for the last spot — what happens?"** `ParkingFloor.allocate` holds a floor-level lock across the scan-then-claim sequence (Java `synchronized`, Python `threading.Lock`), so the second thread's scan simply sees the spot as taken. Per-spot `tryOccupy` is a belt-and-suspenders second check. See [../06-concurrency-essentials.md#java-concurrency-primitives](../06-concurrency-essentials.md#java-concurrency-primitives) for the `ReentrantLock`-with-`tryLock` variant if you want non-blocking allocation instead.
- **"What if we need subscription/monthly pricing instead of pay-per-visit?"** Add a `SubscriptionPricing implements PricingStrategy` that returns 0 for subscribers and short-circuits at the gate — zero changes to `ParkingLot`, `ParkingFloor`, or any other `PricingStrategy`. This is the OCP payoff of making pricing a Strategy from the start.
- **"How would you support a reservation/waitlist so a specific spot is held for a car arriving in 10 minutes?"** Add a `reserve(spot, vehicle, expiry)` path on `ParkingFloor` that marks a spot `RESERVED` (a third occupancy state, not just free/occupied) and a background sweep that reverts to `FREE` on expiry — this is additive to the existing free/occupied check in `allocate`, not a rewrite.
- **"Multiple lots across a city, need to route a car to whichever lot has space?"** That's a new `ParkingLotLocator` service one layer up, holding references to multiple `ParkingLot` instances and querying availability — confirms the earlier call to *not* Singleton-enforce `ParkingLot`, since this follow-up needs more than one instance to coexist in the same process.
- **"What if a Truck needs 2 adjacent Large spots?"** `ParkingSpot.fits` and `ParkingFloor.allocate` would need to reason about contiguous spot ranges instead of a single spot — flag this as a real model change (spots become allocatable in groups) rather than something the current Strategy/Factory seams absorb for free; worth naming as a scope question up front.

## Common mistakes on this problem

- Hardcoding vehicle→spot-size logic as an `if (type == CAR) ... else if (type == TRUCK)` chain instead of either a Strategy or (as here) enum-with-behavior — instant OCP red flag.
- Building a full `Motorcycle`/`Car`/`Truck`/`Electric` class hierarchy when nothing actually overrides behavior — over-engineering that costs interview time for zero payoff; justify subclassing only when behavior, not data, differs.
- Enforcing Singleton on `ParkingLot` via a static `getInstance()` "because parking lot problems always use Singleton" — cargo-culting the pattern name without checking whether the constraint (single instance) actually needs language-level enforcement vs. composition-root discipline.
- Forgetting to synchronize spot allocation at all, then hand-waving "assume single-threaded" when the interviewer explicitly asked about concurrent entry gates — if concurrency is in scope, show the lock, don't just promise one.

## Continue

Next: [02-elevator-system.md](02-elevator-system.md)
