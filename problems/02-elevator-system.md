# Elevator System

The state-machine problem. If you model elevator behavior as booleans (`isMoving`, `doorsOpen`) you will contradict yourself within 10 minutes of follow-ups ("can doors be open while moving?" — your booleans don't forbid it, but physics does). An explicit state machine forbids illegal combinations by construction.

## Requirements

- "How many elevators, how many floors?" → **You decide**: N elevators, M floors, both configured at construction; not relevant to the class design beyond sizing collections.
- "Dispatch algorithm — nearest elevator, or something smarter?" → Support nearest-idle-elevator as the default, but make it pluggable — an interviewer will ask "what if two requests come from floors on the way — should a moving elevator pick them up?" (SCAN-style) as a near-certain follow-up.
- "Can a request specify direction (hall button) vs a destination (car button)?" → **You decide**: model both — a `Request` carries a target floor and, for hall calls, a desired direction; car-panel requests only need a target floor which implies direction from current position.
- "Do doors auto-close after a timeout?" → Yes, model this as a real state (`DoorsOpen`) with a timeout transition back to `Moving`/`Idle`, not a fire-and-forget `Thread.sleep`.
- "Overload/weight sensors, fire alarm override, maintenance mode?" → Out of scope for the core design; call out that `OutOfService` would be one more `ElevatorState` implementation if asked.
- "Multiple elevator banks / zones (low-rise vs high-rise)?" → Out of scope; the `DispatchStrategy` seam is exactly where zone-aware routing would plug in later.

**In scope:** multi-elevator dispatch, explicit per-elevator state machine, thread-safe request intake while an elevator is mid-move, floor display panels reacting to position/door changes.

**Out of scope:** weight/capacity limits, fire/emergency override modes, multi-bank zoning, destination-dispatch UI (grouping passengers by destination before boarding).

## Core entities & relationships

```
ElevatorSystem
  ├─ has-a[*] Elevator
  ├─ has-a[1] DispatchStrategy (interface)
  └─ has-a[*] ElevatorObserver (interface)

Elevator
  ├─ has-a[1] ElevatorState (interface — Idle/Moving/DoorsOpen)
  ├─ has-a[1] Direction (enum)
  └─ has-a[*] Request (thread-safe queue)

Request
  └─ has-a[1] target floor (+ optional desired direction, for hall calls)
```

`ElevatorState` is a real class hierarchy behind an interface, not an enum-with-behavior like the parking lot's `VehicleType`. The difference: each elevator state carries genuinely different *transition logic and side effects* — `Idle` just waits for a request, `Moving` advances a floor and decides whether it has arrived, `DoorsOpen` starts a timer and decides where to go next — not just a different data lookup. That's the GoF-State litmus test: reach for it when the *behavior* of "handle a tick" or "handle a new request" differs by state, not merely a constant differs.

`Direction` (`UP`/`DOWN`/`IDLE`) stays a plain enum — it's an attribute the state machine sets, not itself a thing with behavior worth polymorphism over.

## Design patterns applied

- [State](../patterns/03-behavioral-patterns.md#state) — the core lesson of this problem: `Idle` → `Moving` → `DoorsOpen` → (`Moving` or `Idle`) as explicit classes means "doors open while moving" is structurally impossible rather than a bug you hope not to introduce with a stray boolean flip.
- [Strategy](../patterns/03-behavioral-patterns.md#strategy) — `DispatchStrategy` isolates "which elevator answers this call" from the elevators themselves; swapping nearest-idle for a SCAN-style algorithm that also considers already-moving elevators touches only `ElevatorSystem`'s constructor argument.
- [Observer](../patterns/03-behavioral-patterns.md#observer) — floor display panels subscribe to `ElevatorObserver` and react to position/door-state changes pushed by the elevator, instead of polling every elevator every tick.

## Java implementation

```java
enum Direction { UP, DOWN, IDLE }

final class Request {
    final int targetFloor;
    final Direction desiredDirection; // null for car-panel requests (direction implied by position)
    Request(int targetFloor, Direction desiredDirection) {
        this.targetFloor = targetFloor; this.desiredDirection = desiredDirection;
    }
}

interface ElevatorObserver {
    void onPositionChanged(String elevatorId, int floor, Direction direction);
    void onDoorsStateChanged(String elevatorId, boolean open);
}

interface ElevatorState {
    void addRequest(Elevator elevator, Request request);
    void step(Elevator elevator); // advance one simulation tick; may transition elevator's state
}

final class IdleState implements ElevatorState {
    public void addRequest(Elevator elevator, Request request) {
        elevator.enqueue(request);
        elevator.setState(new MovingState());
    }
    public void step(Elevator elevator) { /* nothing queued, nothing to do */ }
}

final class MovingState implements ElevatorState {
    public void addRequest(Elevator elevator, Request request) { elevator.enqueue(request); }
    public void step(Elevator elevator) {
        Integer target = elevator.peekNextFloor();
        if (target == null) { elevator.setState(new IdleState()); elevator.setDirection(Direction.IDLE); return; }
        if (target > elevator.getCurrentFloor()) { elevator.moveTo(elevator.getCurrentFloor() + 1); elevator.setDirection(Direction.UP); }
        else if (target < elevator.getCurrentFloor()) { elevator.moveTo(elevator.getCurrentFloor() - 1); elevator.setDirection(Direction.DOWN); }
        if (target.equals(elevator.getCurrentFloor())) {
            elevator.popNextFloor();
            elevator.setState(new DoorsOpenState());
        }
    }
}

final class DoorsOpenState implements ElevatorState {
    private final long openedAtMillis = System.currentTimeMillis();
    private static final long DOOR_OPEN_MS = 3000;
    public void addRequest(Elevator elevator, Request request) { elevator.enqueue(request); } // accepted, acted on after close
    public void step(Elevator elevator) {
        if (System.currentTimeMillis() - openedAtMillis < DOOR_OPEN_MS) return; // still within dwell time
        elevator.notifyDoors(false);
        elevator.setState(elevator.hasPendingRequests() ? new MovingState() : new IdleState());
    }
}

final class Elevator {
    private final String id;
    private volatile int currentFloor;
    private volatile Direction direction = Direction.IDLE;
    private ElevatorState state = new IdleState();
    private final Queue<Integer> destinations = new PriorityQueue<>(); // simplification: real SCAN keeps separate up/down queues
    private final List<ElevatorObserver> observers;
    private final Object lock = new Object(); // guards state + destinations together — see concurrency follow-up

    Elevator(String id, int startFloor, List<ElevatorObserver> observers) {
        this.id = id; this.currentFloor = startFloor; this.observers = observers;
    }

    void addRequest(Request request) {
        synchronized (lock) { state.addRequest(this, request); }
    }
    void step() {
        synchronized (lock) { state.step(this); }
    }
    void enqueue(Request r) { destinations.add(r.targetFloor); }
    Integer peekNextFloor() { return destinations.peek(); }
    void popNextFloor() { destinations.poll(); }
    boolean hasPendingRequests() { return !destinations.isEmpty(); }
    void setState(ElevatorState s) { this.state = s; }
    void setDirection(Direction d) { this.direction = d; }
    int getCurrentFloor() { return currentFloor; }
    String getId() { return id; }
    void moveTo(int floor) {
        currentFloor = floor;
        observers.forEach(o -> o.onPositionChanged(id, floor, direction));
    }
    void notifyDoors(boolean open) { observers.forEach(o -> o.onDoorsStateChanged(id, open)); }
    int distanceTo(int floor) { return Math.abs(currentFloor - floor); }
    Direction getDirection() { return direction; }
}

interface DispatchStrategy {
    Elevator select(List<Elevator> elevators, Request request);
}

final class NearestElevatorDispatch implements DispatchStrategy {
    public Elevator select(List<Elevator> elevators, Request request) {
        return elevators.stream().min(Comparator.comparingInt(e -> e.distanceTo(request.targetFloor)))
                         .orElseThrow();
    }
}

final class ScanAwareDispatch implements DispatchStrategy { // prefers an elevator already heading the right way
    public Elevator select(List<Elevator> elevators, Request request) {
        return elevators.stream()
            .filter(e -> e.getDirection() == request.desiredDirection || e.getDirection() == Direction.IDLE)
            .min(Comparator.comparingInt(e -> e.distanceTo(request.targetFloor)))
            .orElseGet(() -> new NearestElevatorDispatch().select(elevators, request));
    }
}

final class FloorDisplayPanel implements ElevatorObserver {
    public void onPositionChanged(String elevatorId, int floor, Direction direction) {
        System.out.printf("Elevator %s at floor %d, heading %s%n", elevatorId, floor, direction);
    }
    public void onDoorsStateChanged(String elevatorId, boolean open) {
        System.out.printf("Elevator %s doors %s%n", elevatorId, open ? "OPEN" : "CLOSED");
    }
}

final class ElevatorSystem {
    private final List<Elevator> elevators;
    private final DispatchStrategy dispatchStrategy;
    private final ScheduledExecutorService ticker = Executors.newSingleThreadScheduledExecutor();

    ElevatorSystem(List<Elevator> elevators, DispatchStrategy dispatchStrategy) {
        this.elevators = elevators; this.dispatchStrategy = dispatchStrategy;
        ticker.scheduleAtFixedRate(this::stepAll, 0, 500, TimeUnit.MILLISECONDS);
    }

    void submitRequest(int floor, Direction desiredDirection) { // hall call
        Elevator chosen = dispatchStrategy.select(elevators, new Request(floor, desiredDirection));
        chosen.addRequest(new Request(floor, desiredDirection));
    }

    private void stepAll() { elevators.forEach(Elevator::step); }
}
```

## Python implementation

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from enum import Enum, auto
from threading import Lock, Timer
import heapq, time

class Direction(Enum):
    UP = auto(); DOWN = auto(); IDLE = auto()

@dataclass(frozen=True)
class Request:
    target_floor: int
    desired_direction: Direction | None = None  # None for car-panel requests

class ElevatorObserver(ABC):
    @abstractmethod
    def on_position_changed(self, elevator_id: str, floor: int, direction: Direction) -> None: ...
    @abstractmethod
    def on_doors_state_changed(self, elevator_id: str, open_: bool) -> None: ...

class ElevatorState(ABC):
    @abstractmethod
    def add_request(self, elevator: "Elevator", request: Request) -> None: ...
    @abstractmethod
    def step(self, elevator: "Elevator") -> None: ...

class IdleState(ElevatorState):
    def add_request(self, elevator, request):
        elevator.enqueue(request)
        elevator.state = MovingState()
    def step(self, elevator):
        pass

class MovingState(ElevatorState):
    def add_request(self, elevator, request):
        elevator.enqueue(request)
    def step(self, elevator):
        target = elevator.peek_next_floor()
        if target is None:
            elevator.state = IdleState()
            elevator.direction = Direction.IDLE
            return
        if target > elevator.current_floor:
            elevator.move_to(elevator.current_floor + 1); elevator.direction = Direction.UP
        elif target < elevator.current_floor:
            elevator.move_to(elevator.current_floor - 1); elevator.direction = Direction.DOWN
        if target == elevator.current_floor:
            elevator.pop_next_floor()
            elevator.state = DoorsOpenState()

class DoorsOpenState(ElevatorState):
    DOOR_OPEN_SECONDS = 3
    def __init__(self):
        self._opened_at = time.monotonic()
    def add_request(self, elevator, request):
        elevator.enqueue(request)
    def step(self, elevator):
        if time.monotonic() - self._opened_at < self.DOOR_OPEN_SECONDS:
            return
        elevator.notify_doors(False)
        elevator.state = MovingState() if elevator.has_pending_requests() else IdleState()

class Elevator:
    def __init__(self, elevator_id: str, start_floor: int, observers: list[ElevatorObserver]):
        self.id = elevator_id
        self.current_floor = start_floor
        self.direction = Direction.IDLE
        self.state: ElevatorState = IdleState()
        self._destinations: list[int] = []  # heap; real SCAN keeps separate up/down heaps
        self._observers = observers
        self._lock = Lock()  # guards state + destinations together — see concurrency follow-up

    def add_request(self, request: Request) -> None:
        with self._lock:
            self.state.add_request(self, request)

    def step(self) -> None:
        with self._lock:
            self.state.step(self)

    def enqueue(self, request: Request) -> None:
        heapq.heappush(self._destinations, request.target_floor)

    def peek_next_floor(self) -> int | None:
        return self._destinations[0] if self._destinations else None

    def pop_next_floor(self) -> None:
        heapq.heappop(self._destinations)

    def has_pending_requests(self) -> bool:
        return bool(self._destinations)

    def move_to(self, floor: int) -> None:
        self.current_floor = floor
        for o in self._observers:
            o.on_position_changed(self.id, floor, self.direction)

    def notify_doors(self, open_: bool) -> None:
        for o in self._observers:
            o.on_doors_state_changed(self.id, open_)

    def distance_to(self, floor: int) -> int:
        return abs(self.current_floor - floor)

class DispatchStrategy(ABC):
    @abstractmethod
    def select(self, elevators: list[Elevator], request: Request) -> Elevator: ...

class NearestElevatorDispatch(DispatchStrategy):
    def select(self, elevators, request):
        return min(elevators, key=lambda e: e.distance_to(request.target_floor))

class ScanAwareDispatch(DispatchStrategy):  # prefers an elevator already heading the right way
    def select(self, elevators, request):
        candidates = [e for e in elevators
                      if e.direction in (request.desired_direction, Direction.IDLE)]
        pool = candidates or elevators
        return min(pool, key=lambda e: e.distance_to(request.target_floor))

class FloorDisplayPanel(ElevatorObserver):
    def on_position_changed(self, elevator_id, floor, direction):
        print(f"Elevator {elevator_id} at floor {floor}, heading {direction.name}")
    def on_doors_state_changed(self, elevator_id, open_):
        print(f"Elevator {elevator_id} doors {'OPEN' if open_ else 'CLOSED'}")

class ElevatorSystem:
    def __init__(self, elevators: list[Elevator], dispatch_strategy: DispatchStrategy, tick_seconds: float = 0.5):
        self.elevators = elevators
        self.dispatch_strategy = dispatch_strategy
        self._schedule_tick(tick_seconds)

    def submit_request(self, floor: int, desired_direction: Direction) -> None:  # hall call
        request = Request(floor, desired_direction)
        chosen = self.dispatch_strategy.select(self.elevators, request)
        chosen.add_request(request)

    def _schedule_tick(self, interval: float) -> None:
        for e in self.elevators:
            e.step()
        Timer(interval, self._schedule_tick, args=(interval,)).start()  # simplified periodic tick; a ThreadPoolExecutor is the production equivalent
```

## Sample walkthrough

```python
panel = FloorDisplayPanel()
e1 = Elevator("E1", start_floor=0, observers=[panel])
e2 = Elevator("E2", start_floor=5, observers=[panel])
system = ElevatorSystem([e1, e2], NearestElevatorDispatch(), tick_seconds=1000)  # manual stepping for the demo

system.submit_request(floor=3, desired_direction=Direction.UP)   # dispatched to E1 (closer: |0-3|=3 < |5-3|=2? -> actually E2)
# E2 selected (distance 2 < 3); e2.step() repeatedly: 5->4->3, then DoorsOpenState fires onDoorsStateChanged
for _ in range(10):
    e1.step(); e2.step()
```

## Follow-up questions

- **"Multiple floors send requests while an elevator is mid-move — does a request get dropped?"** No — `addRequest` and `step` both acquire the same per-elevator lock (Java `Object` monitor / Python `threading.Lock`) before touching `state` and the destination heap, so concurrent calls serialize rather than race. For a real deployment, swap the ad-hoc lock for a `BlockingQueue`/`queue.Queue` of incoming requests per elevator and a single consumer thread draining it — see [../06-concurrency-essentials.md#blockingqueue--producerconsumer-eg-elevator-request-queue](../06-concurrency-essentials.md#blockingqueue--producerconsumer-eg-elevator-request-queue) and the Python [`queue.Queue`](../06-concurrency-essentials.md#queuequeue--thread-safe-producerconsumer-eg-elevator-requests) equivalent.
- **"What if dispatch should consider passenger load / not send an already-full elevator?"** Add a `capacity`/`currentLoad` field to `Elevator` and have `DispatchStrategy.select` filter on it — no change to `ElevatorState` or the state machine, since load is a dispatch-time concern, not a per-tick behavior concern.
- **"How do you avoid starving a floor that a SCAN-style algorithm keeps skipping?"** Track request age and add a "boost" rule to `ScanAwareDispatch` (or a new `FairnessAwareDispatch`) that forces pickup once wait time exceeds a threshold — swap the strategy, `Elevator`/`ElevatorSystem` untouched.
- **"Add an emergency/out-of-service mode."** New `OutOfServiceState implements ElevatorState` that rejects `addRequest` (or redirects it to another elevator via the system) and ignores `step` — this is exactly why State beats booleans: one more class, no existing transition logic touched.
- **"Undo/replay of elevator moves for debugging?"** Wrap each `Request` submission as a [Command](../patterns/03-behavioral-patterns.md#command) object with an `execute()`/log entry; the state machine doesn't need to know moves are being recorded — see the chess undo discussion in [problems/04](04-tic-tac-toe-and-chess.md) for the same pattern applied to move history.

## Common mistakes on this problem

- Modeling elevator status as `boolean isMoving` + `boolean doorsOpen` + `int direction` (0/1/-1) instead of an explicit `ElevatorState` hierarchy — this permits illegal states (doors open *and* moving) and every new mode (out-of-service, emergency) means auditing every boolean check across the codebase.
- Coupling dispatch logic directly into `Elevator` (`elevator.shouldIAnswerThisCall(request)`) instead of extracting `DispatchStrategy` — makes it impossible to swap algorithms without touching every elevator instance.
- Using an unbounded, unsynchronized `List<Request>` as the request queue and reading/writing it from both the request-submission thread and the tick thread — a textbook `ConcurrentModificationException` / lost-update bug that interviewers specifically probe for on this problem.
- Treating `Direction` as just UI decoration instead of using it to drive dispatch decisions (an elevator moving up should preferentially pick up other up-calls on its way) — misses the SCAN-algorithm follow-up entirely.

## Continue

Next: [03-vending-machine.md](03-vending-machine.md)
