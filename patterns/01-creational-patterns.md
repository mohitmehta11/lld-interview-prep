# Creational Patterns

Creational patterns answer one question: **who decides which concrete class gets instantiated, and how much construction-time logic does that decision need?** If the answer is "no logic, just `new` it," you don't need a pattern — say so and move on.

---

## Singleton

**Intent:** Guarantee a class has exactly one instance and provide a single global access point to it.

**When to reach for it in LLD:**
- The problem domain itself has exactly one of something *by definition* — one `ParkingLot`, one `ElevatorControlSystem`, one config registry — not "there happens to be one right now."
- You need a single shared coordinator that mediates access to a scarce resource (a connection pool, an ID generator).

**Structure:**
```
Singleton
  ├─ private static field: instance
  ├─ private constructor (blocks external `new`)
  └─ public static getInstance() -> Singleton
```

**Java** — two idiomatic forms:
```java
// Form 1: private constructor + static holder (lazy, thread-safe via classloader guarantees)
public class ParkingLot {
    private static class Holder {
        private static final ParkingLot INSTANCE = new ParkingLot();
    }
    private ParkingLot() { /* init floors, spots */ }
    public static ParkingLot getInstance() { return Holder.INSTANCE; }
}

// Form 2: enum singleton (Effective Java's preferred form — serialization-safe, concise)
public enum IdGenerator {
    INSTANCE;
    private final AtomicLong counter = new AtomicLong();
    public long next() { return counter.incrementAndGet(); }
}
// usage: IdGenerator.INSTANCE.next();
```
The naive `if (instance == null) instance = new Singleton();` is a race under concurrent first-access — if you must lazy-init without the holder idiom, you need **double-checked locking** with a `volatile` field. Don't hand-roll this without saying the word "volatile" out loud; see [../06-concurrency-essentials.md](../06-concurrency-essentials.md) for why the holder idiom sidesteps the problem entirely.

**Python** — module-level instance is the idiomatic equivalent; classic GoF Singleton is rarely written in Python:
```python
# Pythonic: a module IS the singleton (import caches it) — no class needed
# id_generator.py
_counter = 0
def next_id() -> int:
    global _counter
    _counter += 1
    return _counter

# If a class is required (e.g. interview wants OOP shown explicitly):
class ParkingLot:
    _instance: "ParkingLot | None" = None

    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

    def __init__(self):
        if hasattr(self, "_initialized"):
            return
        self._initialized = True
        # init floors, spots
```

**Related principle:** tension with [DIP](../02-solid-principles.md#d--dependency-inversion-principle) — a global `getInstance()` call hides a dependency instead of injecting it, which is exactly what DIP tells you to avoid. Prefer constructing the single instance once at the composition root and passing it in.

**Used in:** [../problems/01-parking-lot.md](../problems/01-parking-lot.md) (one `ParkingLot` coordinating floors/spots), [../problems/02-elevator-system.md](../problems/02-elevator-system.md) (one dispatcher coordinating multiple cars).

**Watch out for:** this is the single most over-named pattern in LLD interviews. Most "there's only one of these" requirements mean "the composition root (`main`) constructs one instance and passes it around," **not** "the class must enforce its own singleton-ness." Enforcing it via `getInstance()` makes the class untestable (can't inject a fresh one per test) and hides a dependency that DIP says should be explicit. Default to "construct once, inject everywhere"; only reach for a real Singleton when the interviewer's problem would break with two instances (e.g. two `IdGenerator`s handing out colliding IDs) and say that reasoning out loud.

---

## Factory Method

**Intent:** Define one creation method that subclasses (or a simple internal branch) override/vary to produce **one product type**, without the caller knowing which concrete class it got.

**When to reach for it in LLD:**
- Requirement language: "create a notification/vehicle/document of a given type" where the caller only cares about the resulting interface, not the concrete class.
- You want to remove `new ConcreteClass()` calls scattered through client code and centralize them behind one seam ([OCP](../02-solid-principles.md#o--openclosed-principle): adding a new type = one new branch or subclass, not edits everywhere `new` was called).

**Structure:**
```
NotificationFactory
  └─ create(type) -> Notification (interface)
                        ├─ EmailNotification
                        ├─ SmsNotification
                        └─ PushNotification
```

**Java**
```java
public interface Notification {
    void send(String message);
}

public class EmailNotification implements Notification {
    public void send(String message) { System.out.println("Email: " + message); }
}
public class SmsNotification implements Notification {
    public void send(String message) { System.out.println("SMS: " + message); }
}

public class NotificationFactory {
    public static Notification create(NotificationType type) {
        return switch (type) {                       // one seam — new type = one new case + one new class
            case EMAIL -> new EmailNotification();
            case SMS -> new SmsNotification();
            case PUSH -> new PushNotification();
        };
    }
}

Notification n = NotificationFactory.create(NotificationType.EMAIL);
n.send("Your order shipped");
```

**Python**
```python
from abc import ABC, abstractmethod
from enum import Enum

class Notification(ABC):
    @abstractmethod
    def send(self, message: str) -> None: ...

class EmailNotification(Notification):
    def send(self, message: str) -> None:
        print(f"Email: {message}")

class SmsNotification(Notification):
    def send(self, message: str) -> None:
        print(f"SMS: {message}")

class NotificationType(Enum):
    EMAIL = "email"
    SMS = "sms"

class NotificationFactory:
    _registry: dict[NotificationType, type[Notification]] = {
        NotificationType.EMAIL: EmailNotification,
        NotificationType.SMS: SmsNotification,
    }

    @classmethod
    def create(cls, kind: NotificationType) -> Notification:
        return cls._registry[kind]()   # dict dispatch avoids the if/elif chain entirely

n = NotificationFactory.create(NotificationType.EMAIL)
n.send("Your order shipped")
```

**Related principle:** direct application of [OCP](../02-solid-principles.md#o--openclosed-principle) — adding `PushNotification` means adding a class and a registry/case entry, never editing existing call sites.

**Used in:** [../problems/10-notification-and-observer-system.md](../problems/10-notification-and-observer-system.md) (channel creation by type), [../problems/01-parking-lot.md](../problems/01-parking-lot.md) (vehicle/spot-size lookup by type).

**Watch out for:** don't build a `Factory` class for a single concrete type with no variation on the horizon — that's ceremony with no payoff. Also don't confuse this with Abstract Factory (below): Factory Method produces **one** product family member per call; if the interviewer's problem needs *several related objects that must be consistent with each other*, that's Abstract Factory, not a bigger Factory Method.

---

## Abstract Factory

**Intent:** Provide an interface for creating **families of related objects** without specifying their concrete classes — and guarantee the objects returned together are mutually compatible.

**When to reach for it in LLD:**
- Requirement language implies a *pairing/bundle* that must stay consistent: "each payment gateway needs its own processor **and** its own refund handler, and they must match" — mixing a Stripe processor with a PayPal refund handler is a bug, not a valid config.
- You're switching an entire "platform"/"provider" at once (all objects from that provider), not picking one object type independently.

**Structure:**
```
PaymentGatewayFactory (interface)
  ├─ createProcessor() -> PaymentProcessor (interface)
  └─ createRefundHandler() -> RefundHandler (interface)

StripeGatewayFactory implements PaymentGatewayFactory
  ├─ createProcessor() -> StripeProcessor
  └─ createRefundHandler() -> StripeRefundHandler

PaypalGatewayFactory implements PaymentGatewayFactory
  ├─ createProcessor() -> PaypalProcessor
  └─ createRefundHandler() -> PaypalRefundHandler
```

**Java**
```java
public interface PaymentProcessor { boolean charge(double amount); }
public interface RefundHandler { boolean refund(String transactionId); }

public interface PaymentGatewayFactory {
    PaymentProcessor createProcessor();
    RefundHandler createRefundHandler();
}

public class StripeGatewayFactory implements PaymentGatewayFactory {
    public PaymentProcessor createProcessor() { return new StripeProcessor(); }
    public RefundHandler createRefundHandler() { return new StripeRefundHandler(); }
}
public class PaypalGatewayFactory implements PaymentGatewayFactory {
    public PaymentProcessor createProcessor() { return new PaypalProcessor(); }
    public RefundHandler createRefundHandler() { return new PaypalRefundHandler(); }
}

// client only ever talks to the family via the factory it was given — can't accidentally
// pair a Stripe processor with a Paypal refund handler
public class CheckoutService {
    private final PaymentProcessor processor;
    private final RefundHandler refundHandler;

    public CheckoutService(PaymentGatewayFactory factory) {
        this.processor = factory.createProcessor();
        this.refundHandler = factory.createRefundHandler();
    }
}
```

**Python** — `ABC` here, since this is a real is-a contract with two required methods and no attractive structural-typing shortcut:
```python
from abc import ABC, abstractmethod

class PaymentProcessor(ABC):
    @abstractmethod
    def charge(self, amount: float) -> bool: ...

class RefundHandler(ABC):
    @abstractmethod
    def refund(self, transaction_id: str) -> bool: ...

class PaymentGatewayFactory(ABC):
    @abstractmethod
    def create_processor(self) -> PaymentProcessor: ...
    @abstractmethod
    def create_refund_handler(self) -> RefundHandler: ...

class StripeGatewayFactory(PaymentGatewayFactory):
    def create_processor(self) -> PaymentProcessor: return StripeProcessor()
    def create_refund_handler(self) -> RefundHandler: return StripeRefundHandler()

class CheckoutService:
    def __init__(self, factory: PaymentGatewayFactory):
        self.processor = factory.create_processor()
        self.refund_handler = factory.create_refund_handler()   # guaranteed to match processor's provider
```

**Related principle:** [DIP](../02-solid-principles.md#d--dependency-inversion-principle) at a family granularity — `CheckoutService` depends only on the two abstractions and the factory abstraction, never on `Stripe*`/`Paypal*` concretes.

**Used in:** [../problems/03-vending-machine.md](../problems/03-vending-machine.md) (if modeling per-brand product/dispenser families), [../problems/06-splitwise-expense-sharing.md](../problems/06-splitwise-expense-sharing.md) (a settlement-provider family: transfer executor + receipt generator per payment rail).

**Watch out for:** if the "family" only ever has one member (you never actually construct a second related object alongside the first), you don't have an Abstract Factory — you have a Factory Method with an unnecessary wrapper. Only escalate to Abstract Factory when the interviewer's problem has a real *pairing* requirement.

---

## Builder

**Intent:** Separate the construction of a complex object from its representation, so the same construction process can produce different configurations, and callers avoid a telescoping constructor or fragile setter sequence.

**When to reach for it in LLD:**
- Requirement language: an object with many fields, several of which are optional, and combinations matter (a `Notification` with recipient/channel/message/priority/retry-policy; a `SearchQuery` with many optional filters).
- You want the constructed object to be immutable but still ergonomic to build.

**Structure:**
```
Notification (immutable result)
  └─ static nested Builder
        ├─ recipient(...) -> Builder
        ├─ channel(...)  -> Builder
        ├─ message(...)  -> Builder
        └─ build()        -> Notification
```

**Java**
```java
public class Notification {
    private final String recipient;
    private final Channel channel;
    private final String message;
    private final Priority priority;   // optional, has a default

    private Notification(Builder b) {
        this.recipient = b.recipient;
        this.channel = b.channel;
        this.message = b.message;
        this.priority = b.priority;
    }

    public static class Builder {
        private String recipient;
        private Channel channel;
        private String message;
        private Priority priority = Priority.NORMAL;   // default before build()

        public Builder recipient(String r) { this.recipient = r; return this; }
        public Builder channel(Channel c) { this.channel = c; return this; }
        public Builder message(String m) { this.message = m; return this; }
        public Builder priority(Priority p) { this.priority = p; return this; }

        public Notification build() {
            if (recipient == null || channel == null) {
                throw new IllegalStateException("recipient and channel are required"); // validate before construction
            }
            return new Notification(this);
        }
    }
}

Notification n = new Notification.Builder()
    .recipient("user@x.com")
    .channel(Channel.EMAIL)
    .message("Your spot is confirmed")
    .build();
```

**Python** — a fluent builder is *possible* but often unidiomatic; a `@dataclass` with keyword defaults covers most "optional params" cases without a separate Builder class. Use a real Builder only when construction needs multi-step validation or a fluent chain adds real clarity:
```python
from dataclasses import dataclass, field

# Usually enough — no Builder needed:
@dataclass
class Notification:
    recipient: str
    channel: "Channel"
    message: str
    priority: "Priority" = field(default="NORMAL")

n = Notification(recipient="user@x.com", channel=Channel.EMAIL, message="Your spot is confirmed")

# Reach for an explicit Builder only when steps need validation/ordering, e.g.:
class NotificationBuilder:
    def __init__(self) -> None:
        self._recipient: str | None = None
        self._channel: "Channel | None" = None
        self._message: str | None = None
        self._priority: "Priority" = Priority.NORMAL

    def recipient(self, recipient: str) -> "NotificationBuilder":
        self._recipient = recipient
        return self

    def channel(self, channel: "Channel") -> "NotificationBuilder":
        self._channel = channel
        return self

    def build(self) -> Notification:
        if not self._recipient or not self._channel:
            raise ValueError("recipient and channel are required")
        return Notification(self._recipient, self._channel, self._message or "", self._priority)
```

**Related principle:** supports [SRP](../02-solid-principles.md#s--single-responsibility-principle) — construction/validation logic lives in the Builder, not bloating the value object's own class with setter-sequencing rules.

**Used in:** [../problems/09-logging-framework.md](../problems/09-logging-framework.md) (`LogRecord` built with many optional fields), [../problems/10-notification-and-observer-system.md](../problems/10-notification-and-observer-system.md) (multi-channel notification construction).

**Watch out for:** in Python especially, don't build a fluent `Builder` class for an object with 3 required fields and no optional combinatorics — a `@dataclass` constructor call already does that job. Reach for Builder when there's real construction complexity (validation ordering, optional-field combinatorics), not just "this class has more than one field."

---

## Prototype

**Intent:** Create new objects by cloning an existing, fully-configured instance instead of constructing from scratch, when construction is expensive or the "template" configuration is easier to copy than to rebuild.

**When to reach for it in LLD:**
- Requirement language: "simulate a move without affecting the real game state," "duplicate a configured template per new instance" (e.g. a seat-layout template reused per showtime).
- Object graphs that are expensive/awkward to reconstruct field-by-field but trivial to deep-copy.

**Structure:**
```
Board (prototype interface)
  └─ clone() -> Board

ChessBoard implements Board
  └─ clone() -> deep copy of piece grid, for AI move simulation
```

**Java**
```java
public interface Prototype<T> {
    T clone();
}

public class ChessBoard implements Prototype<ChessBoard> {
    private final Piece[][] grid;

    public ChessBoard(Piece[][] grid) { this.grid = grid; }

    @Override
    public ChessBoard clone() {
        Piece[][] copy = new Piece[grid.length][];
        for (int i = 0; i < grid.length; i++) {
            copy[i] = grid[i].clone();   // shallow row clone; deep-clone Piece too if Piece is mutable
        }
        return new ChessBoard(copy);
    }
}

ChessBoard simulated = currentBoard.clone();
simulated.applyMove(candidateMove);   // mutate the clone freely, real board untouched
```

**Python** — `copy.deepcopy` (or a hand-rolled `clone()` when you need control over what's shared vs copied) is the idiomatic equivalent; Python rarely needs a formal `Prototype` interface:
```python
import copy
from dataclasses import dataclass

@dataclass
class ChessBoard:
    grid: list[list["Piece | None"]]

    def clone(self) -> "ChessBoard":
        return copy.deepcopy(self)   # idiomatic: no hand-written field-by-field copy needed

simulated = current_board.clone()
simulated.apply_move(candidate_move)   # real board untouched
```

**Related principle:** loosely supports [OCP](../02-solid-principles.md#o--openclosed-principle) — new "kinds" of board/template can override `clone()` without touching client simulation code, but the bigger win here is just avoiding expensive/error-prone reconstruction.

**Used in:** [../problems/04-tic-tac-toe-and-chess.md](../problems/04-tic-tac-toe-and-chess.md) (clone board to simulate candidate moves before committing), [../problems/07-movie-ticket-booking.md](../problems/07-movie-ticket-booking.md) (clone a seat-layout template per new showtime).

**Watch out for:** this is the lowest-frequency creational pattern in LLD interviews — don't force it in. If plain construction (`new Board()` from a config) is just as cheap and clear as cloning, skip Prototype; only reach for it when reconstruction is genuinely expensive or when a "same-shape template, independent copies" requirement is explicit.

## Continue

Next: [02-structural-patterns.md](02-structural-patterns.md)
