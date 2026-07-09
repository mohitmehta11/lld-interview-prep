# Concurrency Essentials for LLD

LLD interviews rarely *require* a fully thread-safe design up front, but the follow-up almost always arrives: "what if two requests hit this at the same time?" This file is the primitives; the application shows up concretely in [problems/01-parking-lot.md](problems/01-parking-lot.md), [problems/02-elevator-system.md](problems/02-elevator-system.md), and especially [problems/07-movie-ticket-booking.md](problems/07-movie-ticket-booking.md) (the seat-locking showcase).

## The two questions to answer before reaching for any primitive

1. **What is the shared mutable state?** (a `Map<SpotId, Spot>`, a seat's status field, a counter)
2. **What is the smallest critical section that protects it?** Lock the smallest region that preserves correctness — locking too broadly kills throughput, locking too narrowly reintroduces races.

## Java concurrency primitives

### `synchronized` — the simplest tool, method or block level

```java
public class ParkingSpot {
    private boolean occupied;

    public synchronized boolean tryPark(Vehicle v) {   // whole method is the critical section
        if (occupied) return false;
        occupied = true;
        return true;
    }
}
```
Locks on `this` for instance methods (or the `Class` object for static methods). Simple, but coarse — locks the *whole* object even if only one field needs protecting, and you can't `tryLock()` or set a timeout.

### `ReentrantLock` — when you need `tryLock`, timeouts, or fairness

```java
import java.util.concurrent.locks.ReentrantLock;

public class Seat {
    private final ReentrantLock lock = new ReentrantLock();
    private SeatStatus status = SeatStatus.AVAILABLE;

    public boolean tryReserve() {
        if (!lock.tryLock()) return false;         // non-blocking attempt — no other thread waits on us
        try {
            if (status != SeatStatus.AVAILABLE) return false;
            status = SeatStatus.LOCKED;
            return true;
        } finally {
            lock.unlock();                          // ALWAYS unlock in finally
        }
    }
}
```
This is the exact mechanism the seat-locking flow in [problems/07-movie-ticket-booking.md](problems/07-movie-ticket-booking.md) uses — one lock per `Seat`, not one lock for the whole theater (fine-grained locking = higher throughput).

### `ConcurrentHashMap` — thread-safe map without a manual lock

```java
Map<String, ParkingSpot> spotsById = new ConcurrentHashMap<>();
spotsById.computeIfAbsent("A1", id -> new ParkingSpot(id));   // atomic check-and-insert, no external lock needed
```
Default choice whenever the shared state is "a map keyed by ID" (spots, seats, tickets) — avoids a global lock around a `HashMap`.

### `AtomicInteger`/`AtomicLong` — lock-free counters

```java
private final AtomicInteger availableSpots = new AtomicInteger(100);
availableSpots.decrementAndGet();   // atomic, no lock needed for a single counter
```

### `ExecutorService` — thread pools for async work

```java
ExecutorService pool = Executors.newFixedThreadPool(4);
pool.submit(() -> processBooking(seat));

// Timed auto-release of a seat lock (movie booking follow-up)
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
scheduler.schedule(() -> releaseSeatIfStillLocked(seat), 10, TimeUnit.MINUTES);
```

### `BlockingQueue` — producer/consumer, e.g. elevator request queue

```java
BlockingQueue<ElevatorRequest> requests = new LinkedBlockingQueue<>();
requests.put(new ElevatorRequest(floor, direction));   // blocks if full (bounded queue)
ElevatorRequest next = requests.take();                // blocks until an item is available
```

## Python concurrency primitives

### The GIL — say this out loud when asked about Python concurrency

Python's Global Interpreter Lock means only one thread executes Python bytecode at a time, even with multiple threads. Practical implications to state in an interview:
- `threading` is still correct for **I/O-bound** concurrency (waiting on locks, network, disk) and is what you'd use to model "two users hitting the booking system concurrently" in a design discussion.
- For **CPU-bound** parallelism you'd reach for `multiprocessing` instead — but LLD problems are almost always I/O/coordination-bound (locking a seat, not crunching numbers), so `threading` + `Lock` is the right answer to give.
- You still need explicit locks even under the GIL, because the GIL only guarantees atomicity of individual bytecode instructions, not multi-step operations like "check status, then set status" (a classic race even in Python).

### `threading.Lock` — the direct equivalent of Java's `synchronized`/`ReentrantLock`

```python
import threading

class Seat:
    def __init__(self):
        self._lock = threading.Lock()
        self.status = SeatStatus.AVAILABLE

    def try_reserve(self) -> bool:
        if not self._lock.acquire(blocking=False):   # equivalent to Java's tryLock()
            return False
        try:
            if self.status != SeatStatus.AVAILABLE:
                return False
            self.status = SeatStatus.LOCKED
            return True
        finally:
            self._lock.release()

    # idiomatic alternative using the context-manager form when you don't need tryLock semantics
    def reserve_blocking(self) -> None:
        with self._lock:
            self.status = SeatStatus.LOCKED
```

### `threading.RLock` — reentrant lock (same thread can re-acquire)

Use when a locked method might call another method that also acquires the same lock (recursive locking) — direct equivalent of Java's `ReentrantLock` re-entrancy guarantee (plain `synchronized` in Java is also reentrant by default, which is a subtle asymmetry worth knowing: Java's built-in monitor lock is reentrant already, Python's plain `Lock` is NOT, hence `RLock` exists as a separate type).

### `concurrent.futures.ThreadPoolExecutor` — Java's `ExecutorService` equivalent

```python
from concurrent.futures import ThreadPoolExecutor

pool = ThreadPoolExecutor(max_workers=4)
pool.submit(process_booking, seat)
```

### Timed auto-release (movie booking follow-up)

```python
import threading

def release_seat_if_still_locked(seat):
    with seat._lock:
        if seat.status == SeatStatus.LOCKED:
            seat.status = SeatStatus.AVAILABLE

timer = threading.Timer(600, release_seat_if_still_locked, args=[seat])  # 10 min, like Java's ScheduledExecutorService
timer.start()
```

### `queue.Queue` — thread-safe producer/consumer, e.g. elevator requests

```python
import queue
requests: "queue.Queue" = queue.Queue()
requests.put(ElevatorRequest(floor, direction))   # thread-safe, blocks if bounded and full
next_request = requests.get()                     # blocks until available
```

## Deadlock — the follow-up trap

If your design has a scenario where a single operation needs to lock **two** resources (e.g., a transfer between two accounts, or swapping two parking spots), always acquire locks in a **consistent global order** (e.g., by ID) to avoid deadlock:

```java
// consistent ordering by ID prevents A-locks-1-waits-2 / B-locks-2-waits-1 deadlock
Account first = a.getId().compareTo(b.getId()) < 0 ? a : b;
Account second = first == a ? b : a;
synchronized (first) {
    synchronized (second) {
        // transfer logic
    }
}
```
Mention this unprompted if a problem involves any two-resource operation (splitwise settlement, seat swap) — it's a strong senior signal.

## What to say if you're out of time to fully implement thread-safety

It's fine — and expected — to design single-threaded first and narrate the extension: "I'll design the happy path assuming single-threaded access, and call out the two places I'd add locking: [X] and [Y], using [primitive], because [shared state]." This demonstrates the awareness (scored) without burning your entire time budget implementing it (see the time-budget table in [00-evaluation-framework.md](00-evaluation-framework.md)).

## Continue

Next: [problems/00-approach-framework.md](problems/00-approach-framework.md) to start practicing, or [07-common-mistakes.md](07-common-mistakes.md) if you're doing a final pass.
