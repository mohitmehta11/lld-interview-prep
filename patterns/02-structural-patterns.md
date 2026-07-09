# Structural Patterns

Structural patterns answer: **how do existing objects compose into larger structures without new code depending on their concrete shapes?** Adapter/Facade/Proxy all wrap something; Decorator/Composite build recursive structures; Flyweight/Bridge are about the *cost* and *axes* of that structure. Full depth on the first five (Adapter, Decorator, Composite, Facade, Proxy) — Flyweight and Bridge get a short treatment since they show up far less often.

---

## Adapter

**Intent:** Convert the interface of an existing class into another interface clients expect, without touching the existing class.

**When to reach for it in LLD:**
- Requirement language: "integrate a third-party SDK/payment gateway/legacy system" whose method names/signatures don't match your domain interface.
- You want your domain code to depend on *your* abstraction (per [DIP](../02-solid-principles.md#d--dependency-inversion-principle)), not the vendor's, so swapping vendors later doesn't ripple through business logic.

**Structure:**
```
PaymentMethod (interface, your domain)
  └─ StripeAdapter implements PaymentMethod
        └─ has-a[1] StripeSdkClient (third-party, incompatible interface)
```

**Java**
```java
// Third-party SDK you don't control — different method name, different return type
public class StripeSdkClient {
    public StripeChargeResult submitCharge(int amountCents, String currency) { ... }
}

// Your domain interface
public interface PaymentMethod {
    boolean pay(double amount);
}

// Adapter bridges the mismatch — domain code never sees StripeSdkClient directly
public class StripeAdapter implements PaymentMethod {
    private final StripeSdkClient client;
    public StripeAdapter(StripeSdkClient client) { this.client = client; }

    @Override
    public boolean pay(double amount) {
        StripeChargeResult result = client.submitCharge((int) (amount * 100), "USD");
        return result.isSuccessful();
    }
}

PaymentMethod payment = new StripeAdapter(new StripeSdkClient());
payment.pay(19.99);   // CheckoutService only ever calls pay(), unaware Stripe is behind it
```

**Python** — duck typing means Python often needs *no* Adapter class if the shapes already line up; write one explicitly when the third-party surface is genuinely different:
```python
from abc import ABC, abstractmethod

class PaymentMethod(ABC):
    @abstractmethod
    def pay(self, amount: float) -> bool: ...

class StripeSdkClient:                     # third-party, different shape
    def submit_charge(self, amount_cents: int, currency: str) -> "StripeChargeResult": ...

class StripeAdapter(PaymentMethod):
    def __init__(self, client: StripeSdkClient) -> None:
        self._client = client

    def pay(self, amount: float) -> bool:
        result = self._client.submit_charge(int(amount * 100), "USD")
        return result.successful
```

**Related principle:** [DIP](../02-solid-principles.md#d--dependency-inversion-principle) — the domain depends on `PaymentMethod`, the abstraction it defines; the adapter, not the business logic, absorbs the vendor's concretion.

**Used in:** [../problems/07-movie-ticket-booking.md](../problems/07-movie-ticket-booking.md) (wrapping an external payment gateway SDK), [../problems/06-splitwise-expense-sharing.md](../problems/06-splitwise-expense-sharing.md) (adapting a bank-transfer API to an internal `SettlementMethod`).

**Watch out for:** don't build an Adapter around an interface *you already control* — if you own both sides, just change one side to match. Adapter earns its keep only when one side is external/unowned/legacy.

---

## Decorator

**Intent:** Attach additional responsibilities to an object dynamically by wrapping it, giving a flexible alternative to subclassing for combinable behavior.

**When to reach for it in LLD:**
- Requirement language: "notifications can optionally be logged/retried/encrypted," "add a surcharge for weekend/EV-charging on top of the base fee" — behaviors that **stack in combination**, where subclassing would need one class per combination (`LoggedRetryingEmailNotifier`, `RetryingEmailNotifier`, ... — combinatorial explosion).
- Each added responsibility should be independently addable/removable at construction time.

**Structure:**
```
Notifier (interface)
  ├─ EmailNotifier (concrete base)
  └─ NotifierDecorator (abstract) implements Notifier, has-a[1] Notifier
        ├─ LoggingDecorator
        └─ RetryDecorator
```

**Java**
```java
public interface Notifier {
    void send(String message);
}

public class EmailNotifier implements Notifier {
    public void send(String message) { System.out.println("Email: " + message); }
}

public abstract class NotifierDecorator implements Notifier {
    protected final Notifier wrapped;                 // has-a, not is-a subclass of the concrete notifier
    protected NotifierDecorator(Notifier wrapped) { this.wrapped = wrapped; }
}

public class LoggingDecorator extends NotifierDecorator {
    public LoggingDecorator(Notifier wrapped) { super(wrapped); }
    @Override public void send(String message) {
        System.out.println("[LOG] sending: " + message);
        wrapped.send(message);
    }
}

public class RetryDecorator extends NotifierDecorator {
    private final int maxAttempts;
    public RetryDecorator(Notifier wrapped, int maxAttempts) { super(wrapped); this.maxAttempts = maxAttempts; }
    @Override public void send(String message) {
        for (int i = 0; i < maxAttempts; i++) {
            try { wrapped.send(message); return; } catch (RuntimeException e) { /* retry */ }
        }
    }
}

// stack behaviors by composition, chosen at construction time — no new subclass needed
Notifier notifier = new LoggingDecorator(new RetryDecorator(new EmailNotifier(), 3));
notifier.send("Your order shipped");
```

**Python**
```python
from abc import ABC, abstractmethod

class Notifier(ABC):
    @abstractmethod
    def send(self, message: str) -> None: ...

class EmailNotifier(Notifier):
    def send(self, message: str) -> None:
        print(f"Email: {message}")

class NotifierDecorator(Notifier, ABC):
    def __init__(self, wrapped: Notifier) -> None:
        self._wrapped = wrapped

class LoggingDecorator(NotifierDecorator):
    def send(self, message: str) -> None:
        print(f"[LOG] sending: {message}")
        self._wrapped.send(message)

class RetryDecorator(NotifierDecorator):
    def __init__(self, wrapped: Notifier, max_attempts: int) -> None:
        super().__init__(wrapped)
        self._max_attempts = max_attempts

    def send(self, message: str) -> None:
        for _ in range(self._max_attempts):
            try:
                self._wrapped.send(message)
                return
            except RuntimeError:
                continue

notifier = LoggingDecorator(RetryDecorator(EmailNotifier(), max_attempts=3))
notifier.send("Your order shipped")
```
(Python also has `@decorator` function/method decorators for the same *idea* at the function level — mention the distinction if asked: GoF Decorator wraps objects/interfaces, Python `@decorator` syntax wraps callables. Don't conflate them out loud.)

**Related principle:** [OCP](../02-solid-principles.md#o--openclosed-principle) — new combinable behavior is a new decorator class, existing notifiers and decorators are untouched.

**Used in:** [../problems/10-notification-and-observer-system.md](../problems/10-notification-and-observer-system.md) (stackable logging/retry/priority behavior on channels), [../problems/01-parking-lot.md](../problems/01-parking-lot.md) (fee surcharges layered on a base `PricingStrategy`).

**Watch out for:** if there's only ever one or two fixed combinations and they never mix independently, plain composition or a couple of concrete classes is simpler than a Decorator hierarchy — don't reach for it just because "behavior gets added" once.

---

## Composite

**Intent:** Compose objects into tree structures and let clients treat an individual object (leaf) and a group of objects (composite) through the same interface.

**When to reach for it in LLD:**
- Requirement language: "a category can contain books or sub-categories," "a folder contains files or other folders," "a notification group can contain individual recipients or nested groups" — recursive part-whole structures where client code shouldn't branch on "is this a leaf or a group?"

**Structure:**
```
FileSystemEntry (interface)
  ├─ File (leaf) — size(): own size
  └─ Directory (composite) — size(): sum of children's size(), has-a[*] FileSystemEntry
```

**Java**
```java
public interface FileSystemEntry {
    long size();
}

public class File implements FileSystemEntry {
    private final long sizeBytes;
    public File(long sizeBytes) { this.sizeBytes = sizeBytes; }
    @Override public long size() { return sizeBytes; }
}

public class Directory implements FileSystemEntry {
    private final List<FileSystemEntry> children = new ArrayList<>();
    public void add(FileSystemEntry entry) { children.add(entry); }
    @Override public long size() {
        return children.stream().mapToLong(FileSystemEntry::size).sum();   // recurses transparently into sub-directories
    }
}

Directory root = new Directory();
root.add(new File(100));
Directory sub = new Directory();
sub.add(new File(50));
root.add(sub);
root.size();   // 150 — caller never checks File vs Directory
```

**Python**
```python
from abc import ABC, abstractmethod

class FileSystemEntry(ABC):
    @abstractmethod
    def size(self) -> int: ...

class File(FileSystemEntry):
    def __init__(self, size_bytes: int) -> None:
        self._size_bytes = size_bytes
    def size(self) -> int:
        return self._size_bytes

class Directory(FileSystemEntry):
    def __init__(self) -> None:
        self._children: list[FileSystemEntry] = []
    def add(self, entry: FileSystemEntry) -> None:
        self._children.append(entry)
    def size(self) -> int:
        return sum(child.size() for child in self._children)   # uniform recursion, no isinstance() branching
```

**Related principle:** [LSP](../02-solid-principles.md#l--liskov-substitution-principle) — `Directory` and `File` must both honor the `FileSystemEntry` contract fully; the moment `Directory.size()` needs a special case that `File` doesn't, the abstraction is leaking.

**Used in:** [../problems/08-library-management-system.md](../problems/08-library-management-system.md) (category tree containing books or sub-categories), [../problems/06-splitwise-expense-sharing.md](../problems/06-splitwise-expense-sharing.md) (an expense group containing individual expenses or nested sub-groups).

**Watch out for:** don't force Composite onto a structure that's really just one flat collection with no recursive containment — if groups never nest inside groups, you don't need the pattern, a `List<Leaf>` is enough.

---

## Facade

**Intent:** Provide a single, simplified interface to a set of interfaces in a subsystem, so clients don't need to know or orchestrate the subsystem's internals.

**When to reach for it in LLD:**
- Requirement language: a use case that spans several already-well-factored subsystems (seat locking + payment + notification; spot search + ticket issuance + payment) and you want one clean call for the "happy path" orchestration without collapsing those subsystems into one God Object.

**Structure:**
```
BookingFacade
  ├─ uses-a SeatLockService
  ├─ uses-a PaymentMethod
  └─ uses-a NotificationService

Client → BookingFacade.bookSeat(...)   // one call, three subsystems orchestrated behind it
```

**Java**
```java
public class BookingFacade {
    private final SeatLockService seatLockService;
    private final PaymentMethod paymentMethod;
    private final NotificationService notificationService;

    public BookingFacade(SeatLockService s, PaymentMethod p, NotificationService n) {
        this.seatLockService = s; this.paymentMethod = p; this.notificationService = n;
    }

    public boolean bookSeat(String seatId, double price, String userEmail) {
        if (!seatLockService.lock(seatId)) return false;         // step 1: subsystem 1
        try {
            if (!paymentMethod.pay(price)) { seatLockService.unlock(seatId); return false; } // step 2
            notificationService.send(userEmail, "Seat " + seatId + " booked!");              // step 3
            return true;
        } finally {
            seatLockService.unlock(seatId);
        }
    }
}

// client's view: one call, no need to know locking/payment/notification exist separately
new BookingFacade(seatLockService, paymentMethod, notificationService).bookSeat("A1", 15.0, "u@x.com");
```

**Python**
```python
class BookingFacade:
    def __init__(
        self,
        seat_lock_service: "SeatLockService",
        payment_method: "PaymentMethod",
        notification_service: "NotificationService",
    ) -> None:
        self._seat_lock_service = seat_lock_service
        self._payment_method = payment_method
        self._notification_service = notification_service

    def book_seat(self, seat_id: str, price: float, user_email: str) -> bool:
        if not self._seat_lock_service.lock(seat_id):
            return False
        try:
            if not self._payment_method.pay(price):
                return False
            self._notification_service.send(user_email, f"Seat {seat_id} booked!")
            return True
        finally:
            self._seat_lock_service.unlock(seat_id)
```

**Related principle:** [SRP](../02-solid-principles.md#s--single-responsibility-principle) at the orchestration layer — the Facade's one reason to change is "the booking *workflow* changed," while each subsystem still changes only for its own reason.

**Used in:** [../problems/07-movie-ticket-booking.md](../problems/07-movie-ticket-booking.md) (booking workflow across locking/payment/notification), [../problems/01-parking-lot.md](../problems/01-parking-lot.md) (`ParkingLot` facade over spot search + ticketing + pricing).

**Watch out for:** a Facade that just forwards one call to one subsystem method is a pointless indirection — it earns its place only when it's genuinely hiding multi-step orchestration across 2+ subsystems.

---

## Proxy

**Intent:** Provide a surrogate/placeholder for another object to control access to it — lazy instantiation, access control, caching, or logging, transparently to the caller.

**When to reach for it in LLD:**
- Requirement language: "expensive to construct, load on demand" (virtual proxy), "only certain roles can call this" (protection proxy), "cache repeated lookups" (caching proxy), "log/audit every call" (logging proxy).
- The caller should keep using the *same interface* it already had — the proxy is a drop-in.

**Structure:**
```
BookCatalog (interface)
  ├─ RealBookCatalog — hits the DB, slow
  └─ CachingBookCatalogProxy implements BookCatalog, has-a[1] RealBookCatalog
        └─ search(query): check cache first, else delegate + populate cache
```

**Java**
```java
public interface BookCatalog {
    List<Book> search(String query);
}

public class RealBookCatalog implements BookCatalog {
    @Override public List<Book> search(String query) {
        // simulate an expensive DB/full-text search
        return runExpensiveDbQuery(query);
    }
}

public class CachingBookCatalogProxy implements BookCatalog {
    private final RealBookCatalog real;
    private final Map<String, List<Book>> cache = new HashMap<>();

    public CachingBookCatalogProxy(RealBookCatalog real) { this.real = real; }

    @Override public List<Book> search(String query) {
        return cache.computeIfAbsent(query, real::search);   // caller can't tell caching happened
    }
}

BookCatalog catalog = new CachingBookCatalogProxy(new RealBookCatalog());
catalog.search("dune");   // first call: hits DB
catalog.search("dune");   // second call: cache hit, same interface either way
```

**Python**
```python
from abc import ABC, abstractmethod

class BookCatalog(ABC):
    @abstractmethod
    def search(self, query: str) -> list["Book"]: ...

class RealBookCatalog(BookCatalog):
    def search(self, query: str) -> list["Book"]:
        return run_expensive_db_query(query)

class CachingBookCatalogProxy(BookCatalog):
    def __init__(self, real: RealBookCatalog) -> None:
        self._real = real
        self._cache: dict[str, list["Book"]] = {}

    def search(self, query: str) -> list["Book"]:
        if query not in self._cache:
            self._cache[query] = self._real.search(query)
        return self._cache[query]
```

**Related principle:** [OCP](../02-solid-principles.md#o--openclosed-principle)/[DIP](../02-solid-principles.md#d--dependency-inversion-principle) — callers depend on `BookCatalog`, so swapping the real implementation for a caching/protection wrapper requires zero changes at call sites.

**Used in:** [../problems/08-library-management-system.md](../problems/08-library-management-system.md) (caching proxy over catalog search), [../problems/05-lru-cache-and-rate-limiter.md](../problems/05-lru-cache-and-rate-limiter.md) (a rate-limiting proxy in front of an API handler).

**Watch out for:** don't confuse Proxy with Decorator — they look identical in code (both wrap-and-delegate). The distinction is *intent*: Proxy controls **access** to the same conceptual operation (cache/lazy-load/guard); Decorator **adds new behavior/responsibility** on top. If you're stacking multiple independent behaviors, say Decorator; if you're gatekeeping one thing, say Proxy — and don't feel obligated to name either if a plain method call suffices.

---

## Flyweight

**Intent:** Share immutable, fine-grained state across many logical objects to cut memory, instead of duplicating it per instance.

**When to reach for it in LLD:** many objects share identical, expensive-ish immutable data — e.g. every white pawn on a chess board shares the same movement rules/icon metadata; only position is per-instance.

**Java**
```java
public class PieceType {                          // shared, immutable — the flyweight
    private static final Map<String, PieceType> CACHE = new HashMap<>();
    final String name; final String movementRule;
    private PieceType(String name, String rule) { this.name = name; this.movementRule = rule; }
    public static PieceType of(String name, String rule) {
        return CACHE.computeIfAbsent(name, n -> new PieceType(n, rule));  // reuse, don't reallocate
    }
}
public class Piece {
    private final PieceType type;   // shared reference, not a copy
    private Position position;      // per-instance, extrinsic state
}
```

**Python**
```python
class PieceType:
    _cache: dict[str, "PieceType"] = {}

    def __new__(cls, name: str, movement_rule: str) -> "PieceType":
        if name not in cls._cache:
            cls._cache[name] = super().__new__(cls)
        return cls._cache[name]
```

**Related principle:** doesn't map to one SOLID letter directly — it's a memory-sharing optimization layered on top of whatever hierarchy already exists (often Strategy for the movement rule itself).

**Used in:** [../problems/04-tic-tac-toe-and-chess.md](../problems/04-tic-tac-toe-and-chess.md) (shared piece-type metadata across many piece instances).

**Watch out for:** almost never worth naming unless the interviewer explicitly probes memory footprint at scale — leading with Flyweight in a 45-minute interview reads as trivia, not judgment.

---

## Bridge

**Intent:** Decouple an abstraction from its implementation so the two can vary independently, avoiding a class explosion when two orthogonal dimensions both grow.

**When to reach for it in LLD:** you have two independent hierarchies that would otherwise multiply (`Alert`/`Report` × `Email`/`SMS` → 4 classes today, N×M as either side grows) — split into an abstraction hierarchy that *holds* an implementor hierarchy instead of inheriting from it.

**Java**
```java
public interface DeliveryChannel { void deliver(String content); }   // implementor hierarchy
public class EmailChannel implements DeliveryChannel { public void deliver(String c) { /*...*/ } }
public class SmsChannel implements DeliveryChannel { public void deliver(String c) { /*...*/ } }

public abstract class Notification {                                  // abstraction hierarchy
    protected final DeliveryChannel channel;                          // bridge: composition, not inheritance
    protected Notification(DeliveryChannel channel) { this.channel = channel; }
    public abstract void send();
}
public class AlertNotification extends Notification {
    public AlertNotification(DeliveryChannel c) { super(c); }
    public void send() { channel.deliver("ALERT: system down"); }
}
```

**Python**
```python
class DeliveryChannel(ABC):
    @abstractmethod
    def deliver(self, content: str) -> None: ...

class Notification(ABC):
    def __init__(self, channel: DeliveryChannel) -> None:
        self._channel = channel   # bridge: composed, so Notification-kind and channel-kind vary independently
    @abstractmethod
    def send(self) -> None: ...
```

**Related principle:** [OCP](../02-solid-principles.md#o--openclosed-principle) on two axes at once — new notification kinds and new channels are both additive.

**Used in:** [../problems/10-notification-and-observer-system.md](../problems/10-notification-and-observer-system.md) (notification kind × delivery channel, kept independent).

**Watch out for:** if only one of the two dimensions actually varies in the problem, you have a plain Strategy, not a Bridge — don't reach for the heavier pattern name for a one-axis problem.

## Continue

Next: [03-behavioral-patterns.md](03-behavioral-patterns.md)
