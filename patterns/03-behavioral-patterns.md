# Behavioral Patterns

Behavioral patterns answer: **how do objects communicate and vary their behavior at runtime without hardcoding the exact algorithm/flow/receiver?** This is the largest pattern family and the one interviewers reach for most, because most LLD problems are really "model behavior that varies," not "model data that varies."

**Prioritize Strategy, Observer, State, Command, Chain of Responsibility if short on time — the rest are lower-frequency in practice.**

---

## Strategy

**Intent:** Define a family of interchangeable algorithms/policies behind one interface, and let the client pick/inject the one it needs at runtime.

**When to reach for it in LLD:**
- Requirement language: "fee/pricing/discount/split **differs by** type," "support multiple payment/notification/sorting methods" — the canonical "varies by type" tell.
- You catch yourself about to write `if (type == X) ... else if (type == Y) ...` on behavior (not just data).

**Structure:**
```
PricingStrategy (interface)
  ├─ HourlyPricingStrategy
  └─ FlatRatePricingStrategy

ParkingLot
  └─ has-a[1] PricingStrategy   // injected, swappable without touching ParkingLot
```

**Java**
```java
public interface PricingStrategy {
    double calculateFee(Duration parkedDuration);
}

public class HourlyPricingStrategy implements PricingStrategy {
    public double calculateFee(Duration d) { return d.toHours() * 20; }
}
public class FlatRatePricingStrategy implements PricingStrategy {
    public double calculateFee(Duration d) { return 50; }   // ignores duration entirely — still substitutable
}

public class ParkingLot {
    private final PricingStrategy pricingStrategy;          // injected — see DIP
    public ParkingLot(PricingStrategy pricingStrategy) { this.pricingStrategy = pricingStrategy; }
    public double bill(Duration parked) { return pricingStrategy.calculateFee(parked); }
}

ParkingLot lot = new ParkingLot(new HourlyPricingStrategy());
```

**Python**
```python
from abc import ABC, abstractmethod
from datetime import timedelta

class PricingStrategy(ABC):
    @abstractmethod
    def calculate_fee(self, parked_duration: timedelta) -> float: ...

class HourlyPricingStrategy(PricingStrategy):
    def calculate_fee(self, parked_duration: timedelta) -> float:
        return parked_duration.seconds / 3600 * 20

class FlatRatePricingStrategy(PricingStrategy):
    def calculate_fee(self, parked_duration: timedelta) -> float:
        return 50

class ParkingLot:
    def __init__(self, pricing_strategy: PricingStrategy) -> None:
        self._pricing_strategy = pricing_strategy   # injected, not constructed internally

    def bill(self, parked: timedelta) -> float:
        return self._pricing_strategy.calculate_fee(parked)
```

**Related principle:** Strategy *is* the mechanism for [OCP](../02-solid-principles.md#o--openclosed-principle) — see the direct callout in [../02-solid-principles.md](../02-solid-principles.md#o--openclosed-principle). Constructor injection of the strategy is [DIP](../02-solid-principles.md#d--dependency-inversion-principle) in action.

**Used in:** [../problems/01-parking-lot.md](../problems/01-parking-lot.md) (pricing by vehicle type/duration), [../problems/06-splitwise-expense-sharing.md](../problems/06-splitwise-expense-sharing.md) (equal/percentage/exact split strategies).

**Watch out for:** don't build a `Strategy` interface for behavior that never actually varies in the problem — if there's exactly one pricing rule and no signal a second is coming, a plain method is correct and a Strategy interface is premature abstraction.

---

## Observer

**Intent:** Define a one-to-many dependency so that when one object (the subject) changes state, all registered dependents are notified automatically, without the subject knowing their concrete types.

**When to reach for it in LLD:**
- Requirement language: "notify all subscribers when X happens," "update the display board when a spot's status changes," "multiple channels should react to one event" — a variable, growable set of listeners reacting to one source of truth.

**Structure:**
```
Subject (interface): subscribe(Observer), unsubscribe(Observer), notifyAll()
  └─ ParkingFloor — has-a[*] Observer

Observer (interface): onUpdate(event)
  ├─ DisplayBoard
  └─ MobileAppNotifier
```

**Java**
```java
public interface Observer {
    void onUpdate(String event);
}

public interface Subject {
    void subscribe(Observer o);
    void unsubscribe(Observer o);
}

public class ParkingFloor implements Subject {
    private final List<Observer> observers = new ArrayList<>();
    public void subscribe(Observer o) { observers.add(o); }
    public void unsubscribe(Observer o) { observers.remove(o); }

    public void onSpotFreed(String spotId) {
        String event = "Spot " + spotId + " is now free";
        for (Observer o : observers) o.onUpdate(event);   // subject doesn't know/care what the observers are
    }
}

public class DisplayBoard implements Observer {
    public void onUpdate(String event) { System.out.println("[Board] " + event); }
}

ParkingFloor floor = new ParkingFloor();
floor.subscribe(new DisplayBoard());
floor.onSpotFreed("A1");
```

**Python**
```python
from abc import ABC, abstractmethod

class Observer(ABC):
    @abstractmethod
    def on_update(self, event: str) -> None: ...

class ParkingFloor:
    def __init__(self) -> None:
        self._observers: list[Observer] = []

    def subscribe(self, observer: Observer) -> None:
        self._observers.append(observer)

    def unsubscribe(self, observer: Observer) -> None:
        self._observers.remove(observer)

    def on_spot_freed(self, spot_id: str) -> None:
        event = f"Spot {spot_id} is now free"
        for observer in self._observers:
            observer.on_update(event)

class DisplayBoard(Observer):
    def on_update(self, event: str) -> None:
        print(f"[Board] {event}")
```

**Related principle:** [OCP](../02-solid-principles.md#o--openclosed-principle) — adding a new observer type never touches `ParkingFloor`; also decouples subject from dependents, an [ISP](../02-solid-principles.md#i--interface-segregation-principle)-flavored win since observers only need to implement `on_update`.

**Used in:** [../problems/10-notification-and-observer-system.md](../problems/10-notification-and-observer-system.md) (the canonical pub-sub showcase), [../problems/01-parking-lot.md](../problems/01-parking-lot.md) (display boards reacting to spot-availability changes).

**Watch out for:** if there's exactly one listener and it's never going to grow, a direct method call is simpler than standing up subject/observer interfaces — Observer earns its keep when the *set* of listeners is variable or unknown ahead of time. Also watch notification ordering/failure isolation: one throwing observer shouldn't be able to break delivery to the rest (wrap each call, don't let one bad listener take down the loop).

---

## State

**Intent:** Let an object alter its behavior when its internal state changes, by giving each state its own class and delegating state-dependent behavior to the current state object, instead of branching on a status field everywhere.

**When to reach for it in LLD:**
- Requirement language: the problem explicitly names states and legal transitions between them ("Idle → HasMoney → Dispensing → OutOfStock," "elevator is Idle/Moving/DoorsOpen") and different operations are valid/invalid depending on current state.

**Structure:**
```
VendingMachineState (interface): insertCoin(), selectItem(), dispense()
  ├─ IdleState
  ├─ HasMoneyState
  ├─ DispensingState
  └─ OutOfStockState

VendingMachine
  └─ has-a[1] VendingMachineState (current)   // delegates every call to it, swaps it on transition
```

**Java**
```java
public interface VendingMachineState {
    void insertCoin(VendingMachine machine);
    void selectItem(VendingMachine machine);
    void dispense(VendingMachine machine);
}

public class IdleState implements VendingMachineState {
    public void insertCoin(VendingMachine m) { m.setState(new HasMoneyState()); }
    public void selectItem(VendingMachine m) { throw new IllegalStateException("Insert coin first"); }
    public void dispense(VendingMachine m) { throw new IllegalStateException("Insert coin first"); }
}

public class HasMoneyState implements VendingMachineState {
    public void insertCoin(VendingMachine m) { /* accept extra coin, stay in this state */ }
    public void selectItem(VendingMachine m) { m.setState(new DispensingState()); }
    public void dispense(VendingMachine m) { throw new IllegalStateException("Select an item first"); }
}

public class VendingMachine {
    private VendingMachineState state = new IdleState();   // starts Idle
    public void setState(VendingMachineState state) { this.state = state; }
    public void insertCoin() { state.insertCoin(this); }
    public void selectItem() { state.selectItem(this); }
    public void dispense() { state.dispense(this); }
}
```

**Python**
```python
from abc import ABC, abstractmethod

class VendingMachineState(ABC):
    @abstractmethod
    def insert_coin(self, machine: "VendingMachine") -> None: ...
    @abstractmethod
    def select_item(self, machine: "VendingMachine") -> None: ...

class IdleState(VendingMachineState):
    def insert_coin(self, machine: "VendingMachine") -> None:
        machine.state = HasMoneyState()
    def select_item(self, machine: "VendingMachine") -> None:
        raise RuntimeError("Insert coin first")

class HasMoneyState(VendingMachineState):
    def insert_coin(self, machine: "VendingMachine") -> None:
        pass   # accept extra coin, stay in this state
    def select_item(self, machine: "VendingMachine") -> None:
        machine.state = DispensingState()

class VendingMachine:
    def __init__(self) -> None:
        self.state: VendingMachineState = IdleState()   # current state, swapped on transition

    def insert_coin(self) -> None:
        self.state.insert_coin(self)
    def select_item(self) -> None:
        self.state.select_item(self)
```

**Related principle:** [OCP](../02-solid-principles.md#o--openclosed-principle) — a new state is a new class; [LSP](../02-solid-principles.md#l--liskov-substitution-principle) — every state must honor the full `VendingMachineState` contract, even if a transition throws (throwing is a *documented* part of the contract here, not a silent surprise).

**Used in:** [../problems/03-vending-machine.md](../problems/03-vending-machine.md) (the cleanest State showcase in this set), [../problems/02-elevator-system.md](../problems/02-elevator-system.md) (Idle/Moving/DoorsOpen car states).

**Watch out for:** contrast with enum-with-behavior ([../03-java-oop-essentials.md](../03-java-oop-essentials.md), section 7) — if the states are few, fixed, and transitions are simple, a behavior-carrying `enum` can be less ceremony than a full class-per-state hierarchy. Reach for full State when transition logic is nontrivial or states need to hold per-state data.

---

## Command

**Intent:** Encapsulate a request as an object (receiver + action + params), so requests can be queued, logged, undone, or executed by something that doesn't know the request's concrete details.

**When to reach for it in LLD:**
- Requirement language: "queue requests and process them in order," "support undo," "log every operation for replay/audit" — anywhere "the action itself" needs to be a first-class value, not just an immediate method call.

**Structure:**
```
Command (interface): execute(), undo()
  ├─ InsertCoinCommand
  └─ DispenseItemCommand

ElevatorController
  └─ has-a[*] Command (request queue) — enqueues, dequeues and executes FIFO/by-priority
```

**Java**
```java
public interface Command {
    void execute();
    void undo();
}

public class FloorRequestCommand implements Command {
    private final Elevator elevator;
    private final int targetFloor;
    private int previousFloor;

    public FloorRequestCommand(Elevator elevator, int targetFloor) {
        this.elevator = elevator; this.targetFloor = targetFloor;
    }

    @Override public void execute() {
        previousFloor = elevator.getCurrentFloor();
        elevator.moveTo(targetFloor);
    }
    @Override public void undo() { elevator.moveTo(previousFloor); }
}

public class ElevatorController {
    private final Deque<Command> history = new ArrayDeque<>();
    private final Queue<Command> pending = new LinkedList<>();

    public void submit(Command cmd) { pending.add(cmd); }
    public void processNext() {
        Command cmd = pending.poll();
        if (cmd != null) { cmd.execute(); history.push(cmd); }
    }
    public void undoLast() {
        if (!history.isEmpty()) history.pop().undo();
    }
}
```

**Python**
```python
from abc import ABC, abstractmethod
from collections import deque

class Command(ABC):
    @abstractmethod
    def execute(self) -> None: ...
    @abstractmethod
    def undo(self) -> None: ...

class FloorRequestCommand(Command):
    def __init__(self, elevator: "Elevator", target_floor: int) -> None:
        self._elevator = elevator
        self._target_floor = target_floor
        self._previous_floor: int | None = None

    def execute(self) -> None:
        self._previous_floor = self._elevator.current_floor
        self._elevator.move_to(self._target_floor)

    def undo(self) -> None:
        self._elevator.move_to(self._previous_floor)

class ElevatorController:
    def __init__(self) -> None:
        self._pending: deque[Command] = deque()
        self._history: list[Command] = []

    def submit(self, command: Command) -> None:
        self._pending.append(command)

    def process_next(self) -> None:
        if self._pending:
            command = self._pending.popleft()
            command.execute()
            self._history.append(command)

    def undo_last(self) -> None:
        if self._history:
            self._history.pop().undo()
```

**Related principle:** [SRP](../02-solid-principles.md#s--single-responsibility-principle) — the "what to do" (Command) is separated from "when/how it gets invoked" (the invoker/queue), each with its own reason to change.

**Used in:** [../problems/02-elevator-system.md](../problems/02-elevator-system.md) (floor requests queued and dispatched as commands), [../problems/09-logging-framework.md](../problems/09-logging-framework.md) (log-write-as-command for buffering/async flush).

**Watch out for:** don't wrap a single, immediate, un-undoable, un-queueable method call in a `Command` object "for structure" — if nothing ever queues it, logs it, or undoes it, it's just a method call wearing a costume.

---

## Chain of Responsibility

**Intent:** Give more than one object a chance to handle a request by chaining handlers; each handler decides to handle it, pass it to the next, or both, without the sender knowing which handler will actually process it.

**When to reach for it in LLD:**
- Requirement language: "a log message is only emitted if it meets the configured level, and gets routed to potentially multiple appenders," "a request passes through validation steps, any of which can reject it," "escalate through support tiers."

**Structure:**
```
LogHandler (abstract): setNext(handler), handle(record)
  ├─ DebugHandler
  ├─ InfoHandler
  └─ ErrorHandler
        each: if (canHandle) process(record); always pass to next (unless terminal)
```

**Java**
```java
public abstract class LogHandler {
    protected LogHandler next;
    public LogHandler setNext(LogHandler next) { this.next = next; return this; }

    public final void handle(LogRecord record) {
        if (canHandle(record)) process(record);
        if (next != null) next.handle(record);      // pass along regardless — multiple handlers may all fire
    }

    protected abstract boolean canHandle(LogRecord record);
    protected abstract void process(LogRecord record);
}

public class ConsoleHandler extends LogHandler {
    protected boolean canHandle(LogRecord r) { return r.level().ordinal() >= Level.INFO.ordinal(); }
    protected void process(LogRecord r) { System.out.println(r); }
}
public class FileHandler extends LogHandler {
    protected boolean canHandle(LogRecord r) { return r.level().ordinal() >= Level.ERROR.ordinal(); }
    protected void process(LogRecord r) { /* write to file */ }
}

LogHandler chain = new ConsoleHandler().setNext(new FileHandler());
chain.handle(new LogRecord(Level.ERROR, "disk full"));   // both handlers get a look, each decides independently
```

**Python**
```python
from abc import ABC, abstractmethod

class LogHandler(ABC):
    def __init__(self) -> None:
        self._next: "LogHandler | None" = None

    def set_next(self, next_handler: "LogHandler") -> "LogHandler":
        self._next = next_handler
        return next_handler

    def handle(self, record: "LogRecord") -> None:
        if self._can_handle(record):
            self._process(record)
        if self._next is not None:
            self._next.handle(record)

    @abstractmethod
    def _can_handle(self, record: "LogRecord") -> bool: ...
    @abstractmethod
    def _process(self, record: "LogRecord") -> None: ...

class ConsoleHandler(LogHandler):
    def _can_handle(self, record: "LogRecord") -> bool:
        return record.level >= Level.INFO
    def _process(self, record: "LogRecord") -> None:
        print(record)
```

**Related principle:** [OCP](../02-solid-principles.md#o--openclosed-principle) — new handler = new class spliced into the chain, no edits to existing handlers; [SRP](../02-solid-principles.md#s--single-responsibility-principle) — each handler owns exactly one filter/action decision.

**Used in:** [../problems/09-logging-framework.md](../problems/09-logging-framework.md) (the canonical use case — level filtering + multi-appender routing), [../problems/06-splitwise-expense-sharing.md](../problems/06-splitwise-expense-sharing.md) (a validation chain: auth check → balance check → limit check before settling).

**Watch out for:** be explicit about whether your chain **stops at the first handler that fires** (classic "pass it on only if unhandled") or **always propagates to every handler** (like the logging example above, where multiple appenders may all legitimately fire) — conflating the two is a common in-interview bug. State which one you're building before writing code.

---

## Template Method

**Intent:** Define the skeleton of an algorithm in a base class method, deferring specific steps to subclasses — the *shape* of the algorithm is fixed and closed, individual steps are open.

**When to reach for it in LLD:** requirement language like "all games follow setup → turns-until-win/draw → announce winner, but win-checking differs per game" — a shared flow, varying steps.

**Java**
```java
public abstract class BoardGame {
    public final void play() {                 // template: fixed skeleton, `final` so subclasses can't reorder it
        initialize();
        while (!isGameOver()) { takeTurn(); }
        announceResult();
    }
    protected abstract void initialize();
    protected abstract boolean isGameOver();    // steps subclasses fill in
    protected abstract void takeTurn();
    protected void announceResult() { System.out.println("Game over"); }  // has a sensible default, can override
}

public class TicTacToe extends BoardGame {
    protected void initialize() { /* empty 3x3 board */ }
    protected boolean isGameOver() { return checkWin() || boardFull(); }
    protected void takeTurn() { /* current player marks a cell */ }
    private boolean checkWin() { return false; }
    private boolean boardFull() { return false; }
}
```

**Python**
```python
from abc import ABC, abstractmethod

class BoardGame(ABC):
    def play(self) -> None:                     # template method — fixed shape
        self._initialize()
        while not self._is_game_over():
            self._take_turn()
        self._announce_result()

    @abstractmethod
    def _initialize(self) -> None: ...
    @abstractmethod
    def _is_game_over(self) -> bool: ...
    @abstractmethod
    def _take_turn(self) -> None: ...
    def _announce_result(self) -> None:          # default step, overridable
        print("Game over")
```

**Related principle:** the flip side of [OCP](../02-solid-principles.md#o--openclosed-principle) — here the *algorithm structure* is closed (often literally `final` in Java) while individual *steps* are the open extension points.

**Used in:** [../problems/04-tic-tac-toe-and-chess.md](../problems/04-tic-tac-toe-and-chess.md) (shared turn-based game loop, per-game win/move rules).

**Watch out for:** if subclasses end up overriding almost every step with wildly different logic, there's no real shared skeleton left — that's a sign you want Strategy (compose the varying part) instead of Template Method (inherit and override it).

---

## Iterator

**Intent:** Provide sequential access to elements of a collection without exposing its underlying representation (array, tree, linked structure).

**When to reach for it in LLD:** requirement language like "iterate over search results/catalog entries" where the backing structure might change (list today, paginated/lazy source later) and callers shouldn't care.

**Java** — usually just implement `Iterable<T>`/`Iterator<T>` so callers get `for-each` for free:
```java
public class BookShelf implements Iterable<Book> {
    private final List<Book> books = new ArrayList<>();
    @Override public Iterator<Book> iterator() {
        return books.iterator();   // delegate to List's iterator; write a custom one only if traversal logic is nontrivial
    }
}
for (Book b : bookShelf) { System.out.println(b); }   // caller never sees the List underneath
```

**Python** — implement `__iter__`/`__next__`, or simply use a generator function, which is more idiomatic than a hand-rolled iterator class:
```python
class BookShelf:
    def __init__(self) -> None:
        self._books: list["Book"] = []

    def __iter__(self):
        yield from self._books   # generator — idiomatic Python Iterator, no separate Iterator class needed

for book in book_shelf:
    print(book)
```

**Related principle:** [ISP](../02-solid-principles.md#i--interface-segregation-principle)-adjacent — callers depend only on "give me the next element," not on the collection's full interface.

**Used in:** [../problems/08-library-management-system.md](../problems/08-library-management-system.md) (traversing catalog/search results without exposing storage).

**Watch out for:** in both languages, don't hand-write an Iterator class when the language's native iteration protocol (Java's `Iterable`, Python's generators) already covers it — that's transliterating C++ iterator ceremony where it isn't needed.

---

## Visitor

**Intent:** Add new operations to a stable object structure (often a [Composite](02-structural-patterns.md#composite) tree) without modifying the classes in that structure — the operation "visits" each element via double dispatch.

**When to reach for it in LLD:** you have a fixed hierarchy (rarely changes) but need to keep adding *new, unrelated operations* over it (compute total value, render, export) and don't want every new operation to mean editing every element class.

**Java**
```java
public interface CategoryVisitor {
    void visit(Book book);
    void visit(Category category);
}
public interface CatalogEntry {
    void accept(CategoryVisitor visitor);      // double dispatch: entry picks the right visit() overload
}
public class Book implements CatalogEntry {
    public void accept(CategoryVisitor visitor) { visitor.visit(this); }
}
public class TotalCountVisitor implements CategoryVisitor {
    private int count = 0;
    public void visit(Book book) { count++; }
    public void visit(Category category) { for (CatalogEntry e : category.entries()) e.accept(this); }
    public int getCount() { return count; }
}
```

**Python** — often skipped in favor of `isinstance`/`functools.singledispatch`, since Python doesn't need double dispatch ceremony to add a function over a type union:
```python
from functools import singledispatch

@singledispatch
def describe(entry) -> str: ...

@describe.register
def _(entry: "Book") -> str:
    return f"Book: {entry.title}"

@describe.register
def _(entry: "Category") -> str:
    return f"Category: {entry.name} ({len(entry.entries)} items)"
```

**Related principle:** trades off against [OCP](../02-solid-principles.md#o--openclosed-principle) in the opposite direction from Strategy — Visitor makes adding **operations** easy at the cost of making adding **new element types** hard (every visitor needs a new `visit()` overload). Say this trade-off out loud if you name it.

**Used in:** [../problems/08-library-management-system.md](../problems/08-library-management-system.md) (operations like count/export over a stable category tree).

**Watch out for:** this is the least frequently *correct* pattern in LLD interviews — the element hierarchy has to be genuinely stable and the operations genuinely growing, which is a narrower fit than almost any other pattern here. In Python especially, reach for `singledispatch` before a full Visitor class hierarchy.

---

## Mediator

**Intent:** Centralize how a set of objects communicate through one mediator object, so they reference the mediator instead of each other directly, cutting many-to-many coupling down to many-to-one.

**When to reach for it in LLD:** multiple peer objects (elevator cars, chat participants) would otherwise need direct references to each other to coordinate — introduce one coordinator they all talk to instead.

**Java**
```java
public class ElevatorDispatcher {                       // mediator
    private final List<Elevator> cars;
    public ElevatorDispatcher(List<Elevator> cars) { this.cars = cars; }

    public void requestFloor(int floor, Direction direction) {
        Elevator best = cars.stream()
            .min(Comparator.comparingInt(c -> Math.abs(c.getCurrentFloor() - floor)))
            .orElseThrow();
        best.moveTo(floor);                              // cars never talk to each other directly
    }
}
```

**Python**
```python
class ElevatorDispatcher:
    def __init__(self, cars: list["Elevator"]) -> None:
        self._cars = cars

    def request_floor(self, floor: int, direction: "Direction") -> None:
        best = min(self._cars, key=lambda c: abs(c.current_floor - floor))
        best.move_to(floor)
```

**Related principle:** [SRP](../02-solid-principles.md#s--single-responsibility-principle) — coordination logic (which car answers a request) lives in one place, not smeared across every `Elevator` instance.

**Used in:** [../problems/02-elevator-system.md](../problems/02-elevator-system.md) (dispatcher coordinating multiple cars/floor requests).

**Watch out for:** don't relabel an ordinary service/controller class as "Mediator" just to name a pattern — it's worth naming specifically when the alternative would be peer objects holding direct references to each other; if there was never going to be direct peer coupling, you just have a normal coordinating service.

## Continue

Next: [../problems/00-approach-framework.md](../problems/00-approach-framework.md)
