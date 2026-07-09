# Java OOP Essentials for LLD

This is the priority file. You know OOP conceptually (C++/Python) — this file is entirely about **Java-specific idiom**: the syntax and standard-library conventions an interviewer expects you to reach for automatically. Read this once, then *type* the code blocks yourself instead of just reading them.

## 1. Class basics & access modifiers

```java
public class ParkingSpot {
    private final String id;              // private by default for fields — encapsulation
    private VehicleSize size;
    private Vehicle parkedVehicle;        // null when free

    public ParkingSpot(String id, VehicleSize size) {   // constructor, no return type, name == class name
        this.id = id;                                    // `this.` disambiguates field vs param
        this.size = size;
    }

    public boolean isFree() {
        return parkedVehicle == null;
    }

    public void park(Vehicle v) {
        if (!isFree()) throw new IllegalStateException("Spot already occupied");
        this.parkedVehicle = v;
    }
}
```

Access modifiers, narrowest to widest — **default to `private`, widen only when needed**:
| Modifier | Visible to |
|---|---|
| `private` | same class only |
| *(no modifier / package-private)* | same package |
| `protected` | same package + subclasses (even in other packages) |
| `public` | everyone |

Interview default: fields `private`, methods `public` unless it's a helper only used internally (`private`) or meant for subclasses to override/call (`protected`).

## 2. `abstract class` vs `interface` — the decision interviewers watch for

This single decision is scored more than almost anything else in a Java LLD interview.

| Use `interface` when... | Use `abstract class` when... |
|---|---|
| Pure contract, no shared state | Subclasses share state (fields) or partial implementation |
| Class needs to implement *multiple* of them (Java has no multiple inheritance of classes) | You genuinely have an "is-a" hierarchy with common code to reuse |
| Behavioral seam (Strategy-like: `PaymentMethod`, `PricingStrategy`) | Template Method pattern (shared skeleton, subclasses fill in steps) |

```java
// Interface: pure behavioral contract, no state
public interface PaymentMethod {
    boolean pay(double amount);
}

// Interface can have default/static methods since Java 8 (rarely needed in LLD, but know it exists)
public interface Notifier {
    void notify(String msg);
    default void notifyAll(List<String> msgs) { msgs.forEach(this::notify); } // default method
}

// Abstract class: shared state + shared logic + forces subclasses to fill a gap
public abstract class Vehicle {
    protected final String licensePlate;   // shared state
    protected final VehicleSize size;

    protected Vehicle(String licensePlate, VehicleSize size) {
        this.licensePlate = licensePlate;
        this.size = size;
    }

    public String describe() {             // shared, concrete behavior
        return getType() + " [" + licensePlate + "]";
    }

    public abstract String getType();      // forces every subclass to fill this in
}

public class Car extends Vehicle {
    public Car(String plate) { super(plate, VehicleSize.MEDIUM); }  // `super()` calls parent constructor
    @Override public String getType() { return "Car"; }
}
```

A class can implement many interfaces but extend only one class:
```java
public class Car extends Vehicle implements Payable, Insurable { ... }
```

## 3. Polymorphism: overloading vs overriding (a favorite trick question)

- **Overriding** — subclass redefines a parent's method, same signature, resolved at **runtime** (dynamic dispatch). Use `@Override` always — it's free compiler safety.
- **Overloading** — same method name, *different parameter list*, resolved at **compile time**.

```java
class Calculator {
    int add(int a, int b) { return a + b; }               // overload 1
    double add(double a, double b) { return a + b; }      // overload 2 — different signature
}

class Vehicle {
    void honk() { System.out.println("generic honk"); }
}
class Car extends Vehicle {
    @Override void honk() { System.out.println("beep beep"); }  // override — runtime dispatch
}

Vehicle v = new Car();
v.honk(); // prints "beep beep" — decided at runtime based on actual object type, not declared type
```

## 4. `static` vs instance

`static` belongs to the **class**, not any instance — one copy shared across all instances. Used for: constants, factory methods, counters, utility methods.

```java
public class ParkingLot {
    private static int totalLotsCreated = 0;      // shared across ALL ParkingLot instances
    private final String lotId;

    public ParkingLot() {
        totalLotsCreated++;
        this.lotId = "LOT-" + totalLotsCreated;
    }

    public static ParkingLot createDefault() {     // static factory method — common LLD idiom
        return new ParkingLot();
    }
}
```

**LLD idiom:** static factory methods (`ParkingLot.createDefault()`) or a dedicated `Factory` class are how you implement the [Factory pattern](patterns/01-creational-patterns.md#factory-method) in Java.

## 5. `final` — immutability signal

- `final` field: must be assigned exactly once (constructor or declaration) — use this by default for anything that shouldn't change after construction. This is your primary immutability tool pre-Records.
- `final` class: cannot be subclassed (e.g. utility/value classes).
- `final` method: cannot be overridden.

```java
public final class Money {                      // can't be subclassed
    private final long cents;                    // immutable field
    public Money(long cents) { this.cents = cents; }
    public Money add(Money other) { return new Money(this.cents + other.cents); } // returns new instance, doesn't mutate
}
```

## 6. Records (Java 16+) — fastest way to model immutable value objects

If allowed to use modern Java, **records** are the idiomatic replacement for boilerplate immutable data classes (Java's answer to Python's `@dataclass`):

```java
public record Point(int x, int y) {}   // auto-generates constructor, getters (x(), y()), equals, hashCode, toString

public record Money(long cents) {
    public Money add(Money other) { return new Money(this.cents + other.cents); }  // can still add methods
}
```
If you're unsure the interviewer's Java version supports records, mention it and fall back to a manual immutable class (section 5) — say "I'd normally use a record here for brevity."

## 7. Enums with behavior — not just constants

Java enums are full classes — they can have fields, constructors, methods, and even **per-constant method overrides**. This is a very high-signal Java idiom for LLD (vehicle types, payment states, elevator direction, etc.).

```java
public enum VehicleSize {
    MOTORCYCLE(1), CAR(2), TRUCK(4);        // each constant carries data

    private final int spotsRequired;
    VehicleSize(int spotsRequired) { this.spotsRequired = spotsRequired; }   // enum constructor — implicitly private
    public int getSpotsRequired() { return spotsRequired; }
}

public enum Direction {
    UP {
        @Override public Direction opposite() { return DOWN; }   // per-constant override
    },
    DOWN {
        @Override public Direction opposite() { return UP; }
    };
    public abstract Direction opposite();
}
```
Using an enum here (instead of an `int`/`String` constant or a full class hierarchy) is frequently the *correct minimal* choice when the set of variants is small, fixed, and known at compile time — contrast with Strategy (interface + classes) when variants need injected behavior or are expected to grow/be added externally.

## 8. Generics — write container/utility classes once

```java
public class Pair<A, B> {
    private final A first;
    private final B second;
    public Pair(A first, B second) { this.first = first; this.second = second; }
    public A getFirst() { return first; }
    public B getSecond() { return second; }
}

// Bounded type parameter — T must be Comparable
public class SortedBox<T extends Comparable<T>> {
    private final List<T> items = new ArrayList<>();
    public void add(T item) { items.add(item); Collections.sort(items); }
}

Pair<String, Integer> p = new Pair<>("age", 30);
```
In LLD you'll mostly *use* generics via the Collections framework (`List<Vehicle>`, `Map<String, ParkingSpot>`) rather than writing your own generic classes — but know the syntax so you're not stuck typing `Object` and casting.

## 9. Collections framework — the part you'll use constantly

| Interface | Common impl | LLD use case |
|---|---|---|
| `List<T>` | `ArrayList<>` | ordered collection, e.g. floors in a parking lot |
| `Map<K,V>` | `HashMap<>` | id → object lookup, e.g. `Map<String, ParkingSpot>` |
| `Map<K,V>` (ordered) | `LinkedHashMap<>` | insertion-order map — **the standard LRU cache building block** |
| `Set<T>` | `HashSet<>` | uniqueness, e.g. set of active tickets |
| `Deque<T>` | `ArrayDeque<>` | stack/queue behavior — undo stacks, elevator request queues |
| `Queue<T>` | `LinkedList<>` / `ArrayDeque<>` | FIFO processing |
| `PriorityQueue<T>` | — | scheduling by priority, e.g. nearest elevator, earliest ticket |

```java
Map<String, ParkingSpot> spotsById = new HashMap<>();
spotsById.put("A1", new ParkingSpot("A1", VehicleSize.CAR));

PriorityQueue<ElevatorRequest> requests = new PriorityQueue<>(
    Comparator.comparingInt(ElevatorRequest::getFloor)   // Comparator via method reference
);
```

### `Comparable` vs `Comparator`

- **`Comparable<T>`** — the class defines its own *natural* ordering by implementing `compareTo`. One per class.
- **`Comparator<T>`** — an external, pluggable ordering. Many per class, injected where needed (this is itself a mini-Strategy).

```java
public class ParkingTicket implements Comparable<ParkingTicket> {
    private final LocalDateTime entryTime;
    @Override public int compareTo(ParkingTicket other) {
        return this.entryTime.compareTo(other.entryTime);   // natural order = by entry time
    }
}

// external, alternate ordering without touching ParkingTicket
Comparator<ParkingTicket> byFeeDesc = Comparator.comparingDouble(ParkingTicket::getFee).reversed();
tickets.sort(byFeeDesc);
```

## 10. `equals`, `hashCode`, `toString` — override together, or not at all

If you override `equals`, you **must** override `hashCode` (equal objects must have equal hashcodes, or `HashMap`/`HashSet` silently break). `toString` is separate but always worth overriding for debuggability.

```java
public class Money {
    private final long cents;
    public Money(long cents) { this.cents = cents; }

    @Override public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Money)) return false;
        Money other = (Money) o;
        return this.cents == other.cents;
    }
    @Override public int hashCode() { return Objects.hash(cents); }
    @Override public String toString() { return "$" + (cents / 100.0); }
}
```
(Records generate all three automatically — another reason to reach for them when allowed.)

## 11. Exceptions — checked vs unchecked, and the LLD convention

- **Checked** (`extends Exception`) — caller *must* handle or declare (`throws`). Use for recoverable conditions the caller should plan for.
- **Unchecked** (`extends RuntimeException`) — caller isn't forced to handle. Use for programming errors / invariant violations.

**LLD convention:** define your own small exception hierarchy for domain errors — it reads as senior and gives you a clean place to handle domain-specific failures.

```java
public class SpotNotAvailableException extends RuntimeException {   // unchecked: caller shouldn't be forced to try/catch every park() call
    public SpotNotAvailableException(String message) { super(message); }
}

public void park(Vehicle v) {
    ParkingSpot spot = findAvailableSpot(v)
        .orElseThrow(() -> new SpotNotAvailableException("No spot for " + v.getType()));
    spot.park(v);
}
```

## 12. Constructors, `this()`, `super()`, and the Builder idiom

```java
public class ParkingTicket {
    private final String id;
    private final Vehicle vehicle;
    private final LocalDateTime entryTime;

    public ParkingTicket(String id, Vehicle vehicle) {
        this(id, vehicle, LocalDateTime.now());   // `this(...)` delegates to another constructor
    }
    public ParkingTicket(String id, Vehicle vehicle, LocalDateTime entryTime) {
        this.id = id; this.vehicle = vehicle; this.entryTime = entryTime;
    }
}
```

When a class has many optional/combinatorial fields, use the **Builder pattern** ([details here](patterns/01-creational-patterns.md#builder)) instead of a telescoping constructor:
```java
Notification n = new Notification.Builder()
    .recipient("user@x.com")
    .channel(Channel.EMAIL)
    .message("Your spot is confirmed")
    .build();
```

## 13. Functional interfaces & lambdas (Java 8+) — know these exist, use sparingly in LLD

```java
Comparator<ParkingTicket> byEntry = (t1, t2) -> t1.getEntryTime().compareTo(t2.getEntryTime());  // lambda
Runnable task = () -> System.out.println("run");
Function<Vehicle, Double> feeFn = v -> v.getSize().getSpotsRequired() * 10.0;
```
Don't over-lambda a design meant to demonstrate OOP class structure — an interviewer testing LLD wants to see `interface`/`class` structure, not a stream-heavy functional rewrite. Reach for lambdas mainly for `Comparator`s and simple callbacks.

## 14. `instanceof` pattern matching (Java 16+) and casting

```java
if (vehicle instanceof Truck truck) {          // pattern variable, avoids separate cast
    System.out.println(truck.getTrailerLength());
}
```
Needing `instanceof` checks against concrete types in business logic is usually itself a **polymorphism smell** — if you catch yourself writing `if (x instanceof Car) ... else if (x instanceof Truck)`, ask whether a virtual method call would remove the branch entirely (that's OCP again).

## 15. Nested/inner classes — mainly relevant for Builder & Iterator

```java
public class Notification {
    private final String recipient;
    private Notification(Builder b) { this.recipient = b.recipient; }

    public static class Builder {                       // static nested class — doesn't need an outer instance
        private String recipient;
        public Builder recipient(String r) { this.recipient = r; return this; }
        public Notification build() { return new Notification(this); }
    }
}
```

## Continue

Next: [04-python-oop-essentials.md](04-python-oop-essentials.md) — then [05-java-vs-python-cheatsheet.md](05-java-vs-python-cheatsheet.md) to lock in the side-by-side mapping.
