# SOLID Principles — the highest-leverage topic in LLD

Every pattern in [patterns/](patterns/00-overview.md) and every problem in [problems/](problems/00-approach-framework.md) is, underneath, an application of one or more of these five. Internalize these once, deeply, and everything downstream is just pattern-matching.

The recurring meta-point: SOLID is about **isolating the axis of change**. Every principle answers "if requirement X changes, how many existing files do I have to touch?" The target is always: touch 0 existing files, add 1 new file.

---

## S — Single Responsibility Principle

**A class should have one reason to change.** Not "one method" — one *axis of change*.

**Smell:** a `ParkingTicket` class that also computes pricing, formats a receipt string, and sends an email. Three unrelated reasons to change (fee rules change / receipt format changes / notification mechanism changes) are welded into one class.

**Java**
```java
// BAD: mixes ticket data, pricing, and notification
class ParkingTicket {
    Vehicle vehicle;
    LocalDateTime entryTime;

    double calculateFee(LocalDateTime exitTime) { /* pricing logic */ return 0; }
    void sendReceiptEmail(String email) { /* SMTP logic */ }
}

// GOOD: split by axis of change
class ParkingTicket {
    private final Vehicle vehicle;
    private final LocalDateTime entryTime;
    // just data + basic accessors
}

interface PricingStrategy {
    double calculateFee(ParkingTicket ticket, LocalDateTime exitTime);
}

interface NotificationService {
    void send(String recipient, String message);
}
```

**Python**
```python
# BAD
class ParkingTicket:
    def __init__(self, vehicle, entry_time):
        self.vehicle = vehicle
        self.entry_time = entry_time

    def calculate_fee(self, exit_time): ...
    def send_receipt_email(self, email): ...

# GOOD
@dataclass(frozen=True)
class ParkingTicket:
    vehicle: Vehicle
    entry_time: datetime

class PricingStrategy(ABC):
    @abstractmethod
    def calculate_fee(self, ticket: ParkingTicket, exit_time: datetime) -> float: ...

class NotificationService(ABC):
    @abstractmethod
    def send(self, recipient: str, message: str) -> None: ...
```

---

## O — Open/Closed Principle

**Open for extension, closed for modification.** Adding a new variant should mean *adding a new class*, never editing a switch/if-else on a type.

**Smell:** `if (vehicle.getType() == VehicleType.CAR) fee = ...; else if (... == TRUCK) fee = ...;` — every new vehicle type means editing this method again.

**Java**
```java
// BAD
double calculateFee(Vehicle v) {
    if (v.getType() == VehicleType.CAR) return 20;
    else if (v.getType() == VehicleType.TRUCK) return 50;
    throw new IllegalArgumentException();
}

// GOOD — new vehicle type = new class, this method never changes again
interface FeeCalculator {
    double calculate(Duration parkedDuration);
}

class CarFeeCalculator implements FeeCalculator {
    public double calculate(Duration d) { return d.toHours() * 20; }
}
class TruckFeeCalculator implements FeeCalculator {
    public double calculate(Duration d) { return d.toHours() * 50; }
}
```

**Python**
```python
# GOOD
class FeeCalculator(ABC):
    @abstractmethod
    def calculate(self, parked_duration: timedelta) -> float: ...

class CarFeeCalculator(FeeCalculator):
    def calculate(self, parked_duration): return parked_duration.seconds / 3600 * 20

class TruckFeeCalculator(FeeCalculator):
    def calculate(self, parked_duration): return parked_duration.seconds / 3600 * 50
```

This is literally the [Strategy pattern](patterns/03-behavioral-patterns.md#strategy) — OCP is the principle, Strategy is the mechanism.

---

## L — Liskov Substitution Principle

**Subtypes must be substitutable for their base type without breaking caller expectations.** A subclass shouldn't strengthen preconditions, weaken postconditions, or throw new exceptions the base type's contract didn't promise.

**Classic violation:** `Square extends Rectangle` where `setWidth`/`setHeight` are overridden to keep width==height — this breaks any code that does `rect.setWidth(5); rect.setHeight(10); assert rect.area() == 50`.

**LLD-relevant violation:** a `ReadOnlyAccount extends Account` that overrides `withdraw()` to throw `UnsupportedOperationException`. Any code that polymorphically calls `withdraw()` on an `Account` now has a landmine.

**Java**
```java
// BAD — violates LSP: caller can't treat all Bird as substitutable
class Bird { void fly() {} }
class Penguin extends Bird {
    void fly() { throw new UnsupportedOperationException(); } // surprise!
}

// GOOD — model the actual capability, don't force a false is-a
interface Bird {}
interface FlyingBird extends Bird { void fly(); }
class Sparrow implements FlyingBird { public void fly() {} }
class Penguin implements Bird {} // no fly(), no lie
```

**Python**
```python
# GOOD — same fix: separate the capability from the base type
class Bird(ABC): ...
class FlyingBird(Bird, ABC):
    @abstractmethod
    def fly(self) -> None: ...

class Sparrow(FlyingBird):
    def fly(self): print("flying")

class Penguin(Bird):
    pass  # correctly has no fly()
```

**Interview tell:** if you catch yourself overriding a method just to make it a no-op or throw, that's an LSP red flag — say so out loud and restructure the hierarchy. Interviewers plant this on purpose (e.g., "what if we add a `Motorcycle` that doesn't need a `ParkingSpot` with EV charging" or "what if some `Account` types can't be overdrawn").

---

## I — Interface Segregation Principle

**Don't force a class to implement methods it doesn't need.** Prefer several small, focused interfaces over one fat interface.

**Smell:** one `Worker` interface with `code()`, `attendStandup()`, `deploy()` forced onto every implementer including a `ContractDesigner` who doesn't deploy.

**Java**
```java
// BAD
interface Worker {
    void code();
    void deploy();
    void designUI();
}

// GOOD — segregate by role
interface Coder { void code(); }
interface Deployer { void deploy(); }
interface Designer { void designUI(); }

class BackendEngineer implements Coder, Deployer { ... }
class UIDesigner implements Designer { ... }
```

**Python** — Python's structural typing (`Protocol`) makes ISP almost automatic, but the discipline still matters when using `ABC`:
```python
# GOOD
class Coder(Protocol):
    def code(self) -> None: ...

class Deployer(Protocol):
    def deploy(self) -> None: ...

class BackendEngineer:  # implements both protocols structurally, no inheritance needed
    def code(self): ...
    def deploy(self): ...
```

**LLD-relevant example:** a `Printable` + `Scannable` + `Faxable` split for a multi-function-printer problem, instead of one `MultiFunctionDevice` interface that a plain `Printer` is forced to stub out.

---

## D — Dependency Inversion Principle

**High-level modules should depend on abstractions, not concretions. Concretions depend on abstractions too.** In practice: constructor-inject an interface, never `new` a concrete dependency inside business logic.

**Smell:** `ParkingLot` directly instantiates `CreditCardPayment` inside `processPayment()` — now `ParkingLot` (high-level policy) is coupled to a low-level payment detail, and you can't swap/mock/extend it.

**Java**
```java
// BAD — high-level ParkingLot depends on a concrete low-level class
class ParkingLot {
    private CreditCardPayment payment = new CreditCardPayment(); // hard dependency
}

// GOOD — depend on the abstraction, inject the concretion
interface PaymentMethod {
    boolean pay(double amount);
}

class ParkingLot {
    private final PaymentMethod paymentMethod;
    ParkingLot(PaymentMethod paymentMethod) { this.paymentMethod = paymentMethod; } // constructor injection
}

class CreditCardPayment implements PaymentMethod {
    public boolean pay(double amount) { /* ... */ return true; }
}

// wiring happens at the "top" (main / composition root)
ParkingLot lot = new ParkingLot(new CreditCardPayment());
```

**Python**
```python
# GOOD
class PaymentMethod(ABC):
    @abstractmethod
    def pay(self, amount: float) -> bool: ...

class ParkingLot:
    def __init__(self, payment_method: PaymentMethod):
        self.payment_method = payment_method  # injected, not constructed internally

class CreditCardPayment(PaymentMethod):
    def pay(self, amount: float) -> bool:
        return True

lot = ParkingLot(payment_method=CreditCardPayment())
```

**Why interviewers love probing this:** DIP is what makes your design *testable* and is exactly what enables the "now add X" follow-up (swap in `UPIPayment`, `MockPayment` for tests) without touching `ParkingLot` at all. Always constructor-inject collaborators; never hardcode `new SomeConcreteClass()` inside a class that represents business logic.

---

## The one-sentence version of each (say these out loud in interviews)

- **S**: one class, one axis of change.
- **O**: add a class, don't edit a switch statement.
- **L**: a subtype must honor every promise the base type made.
- **I**: many small role-interfaces beat one fat interface.
- **D**: depend on interfaces, inject concretions at the top.

## Continue

Next: [03-java-oop-essentials.md](03-java-oop-essentials.md) (your priority weak spot) or [04-python-oop-essentials.md](04-python-oop-essentials.md).
