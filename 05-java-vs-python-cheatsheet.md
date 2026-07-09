# Java ↔ Python Side-by-Side Cheatsheet

Purpose: stop mid-interview language context-switch lag. Skim right before switching languages.

## Structure & declarations

| Concept | Java | Python |
|---|---|---|
| Class | `public class Car extends Vehicle {}` | `class Car(Vehicle):` |
| Interface | `interface Payable { boolean pay(double amt); }` | `class Payable(ABC):`<br>`    @abstractmethod`<br>`    def pay(self, amt: float) -> bool: ...` |
| Constructor | `public Car(String plate) { super(plate); }` | `def __init__(self, plate): super().__init__(plate)` |
| Field, private | `private String plate;` | `self._plate` (convention, not enforced) |
| Constant | `public static final int MAX = 10;` | `MAX: Final[int] = 10` (module or class level) |
| Instantiate | `new Car("KA01")` | `Car("KA01")` |
| No return value | `void park() {}` | `def park(self) -> None:` |
| Null / None | `null` | `None` |
| String format | `String.format("%s-%d", a, b)` | `f"{a}-{b}"` |
| Ternary | `x > 0 ? "pos" : "neg"` | `"pos" if x > 0 else "neg"` |

## OOP mechanics

| Concept | Java | Python |
|---|---|---|
| Abstract method | `abstract void foo();` | `@abstractmethod` + `def foo(self): ...` |
| Override | `@Override` annotation (optional, enforced by convention) | no annotation; just redefine (no compiler enforcement) |
| Call parent method | `super.foo()` | `super().foo()` |
| Multiple interfaces | `class X implements A, B` | `class X(A, B)` — Python allows multiple **class** inheritance too |
| Static method | `static Foo create() {}` | `@staticmethod def create():` |
| Class-level factory bound to cls | (use static method) | `@classmethod def create(cls):` |
| Enum | `enum Direction { UP, DOWN }` | `class Direction(Enum): UP = auto(); DOWN = auto()` |
| Equality override | `equals()` + `hashCode()` | `__eq__` + `__hash__` (or `@dataclass`) |
| String representation | `toString()` | `__repr__` / `__str__` |
| Natural ordering | `implements Comparable<T>` + `compareTo()` | `__lt__` (+`total_ordering`) or just `key=` in `sort()` |
| External ordering | `Comparator<T>` object | plain function passed as `key=` |
| Operator overload | not supported (except a few built-in types) | `__add__`, `__eq__`, etc. |
| Immutable value type | `final` fields, or `record Point(int x, int y) {}` | `@dataclass(frozen=True)` |
| Structural typing / duck contract | not really — must declare `implements` | `typing.Protocol` |

## Collections

| Need | Java | Python |
|---|---|---|
| Dynamic array | `List<T> l = new ArrayList<>();` | `l: list[T] = []` |
| Hash map | `Map<K,V> m = new HashMap<>();` | `m: dict[K, V] = {}` |
| Insertion-ordered map | `LinkedHashMap<>` | `dict` (ordered by default 3.7+) / `OrderedDict` for `move_to_end` |
| Set | `Set<T> s = new HashSet<>();` | `s: set[T] = set()` |
| Stack/Deque | `Deque<T> d = new ArrayDeque<>();` | `from collections import deque; d = deque()` |
| Priority queue | `PriorityQueue<T> pq = new PriorityQueue<>(comparator);` | `import heapq; heapq.heappush(pq, (priority, item))` |
| Iterate map | `for (var e : map.entrySet())` | `for k, v in d.items():` |
| Null-safe default | `map.getOrDefault(k, def)` | `d.get(k, default)` |
| Default-on-missing | manual `if (!map.containsKey(k)) map.put(k, new ArrayList<>());` | `from collections import defaultdict; d = defaultdict(list)` |

## Generics / typing

| Java | Python |
|---|---|
| `List<Vehicle> vehicles;` | `vehicles: list[Vehicle]` |
| `<T extends Comparable<T>>` (bounded generic) | `TypeVar('T', bound=Comparable)` (rarely needed explicitly in LLD interviews — just type-hint normally) |
| `Optional<T>` | `Optional[T]` / `T | None` |

```java
Optional<ParkingSpot> spot = findSpot(vehicle);
if (spot.isPresent()) { spot.get().park(vehicle); }
spot.ifPresentOrElse(s -> s.park(vehicle), () -> System.out.println("no spot"));
```
```python
spot: ParkingSpot | None = find_spot(vehicle)
if spot is not None:
    spot.park(vehicle)
```

## Exceptions

| Java | Python |
|---|---|
| `class MyError extends RuntimeException {}` | `class MyError(Exception):` |
| `throw new MyError("msg");` | `raise MyError("msg")` |
| `try { } catch (MyError e) { } finally { }` | `try:` / `except MyError as e:` / `finally:` |
| Checked exceptions (`throws`) | no equivalent — all exceptions are "unchecked" |
| try-with-resources | `try (var r = open()) { ... }` | `with open() as r:` (context manager) |

## Concurrency (see [06-concurrency-essentials.md](06-concurrency-essentials.md) for depth)

| Java | Python |
|---|---|
| `synchronized(this) { ... }` | `with self._lock: ...` (`threading.Lock`) |
| `ReentrantLock` | `threading.RLock` |
| `ExecutorService` | `concurrent.futures.ThreadPoolExecutor` |
| `ConcurrentHashMap` | plain `dict` + explicit lock (or note the GIL nuance) |
| `AtomicInteger` | `itertools.count()` isn't atomic across threads — use a `Lock` around a plain counter |

## The mental translation rule of thumb

- **Java → Python**: drop the type on the left of declarations (keep the type *hint* on the right), drop semicolons/braces → indentation, replace `interface`+`implements` with `ABC`+`abstractmethod` (or `Protocol` if it's purely structural), replace getters/setters with `@property` unless validation-free.
- **Python → Java**: add explicit types everywhere, decide interface vs abstract class deliberately (see [03-java-oop-essentials.md §2](03-java-oop-essentials.md#2-abstract-class-vs-interface--the-decision-interviewers-watch-for)), replace `dict`/`list` literals with `new HashMap<>()`/`new ArrayList<>()`, and remember every checked exception you throw must be declared or caught.

## Continue

Next: [patterns/00-overview.md](patterns/00-overview.md)
