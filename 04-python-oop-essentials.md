# Python OOP Essentials for LLD

You already know Python OOP fundamentals — this is a refresher focused narrowly on the idioms interviewers specifically look for in an LLD round, where "clean Pythonic OOP" means something more specific than "class works."

## 1. Real interfaces: `abc.ABC` (not just convention)

Python has no `interface` keyword — the LLD-idiomatic way to express a Strategy/contract is `ABC` + `@abstractmethod`, which *enforces* that subclasses implement it (instantiating an incomplete subclass raises `TypeError`). Don't just write a base class with `pass` bodies and a comment — that's the Java-abstract-class idiom used wrong in Python and interviewers notice.

```python
from abc import ABC, abstractmethod

class PaymentMethod(ABC):
    @abstractmethod
    def pay(self, amount: float) -> bool: ...

class CreditCardPayment(PaymentMethod):
    def pay(self, amount: float) -> bool:
        print(f"Charging ${amount} to credit card")
        return True

# PaymentMethod() -> TypeError: Can't instantiate abstract class
```

## 2. `Protocol` — structural typing, when you don't need a real hierarchy

`typing.Protocol` gives you [ISP](02-solid-principles.md#i--interface-segregation-principle) almost for free: any class with the matching method signature satisfies the protocol, no inheritance required. Use this when you want to type-hint "anything shaped like X" without forcing unrelated classes into a shared base.

```python
from typing import Protocol

class Notifiable(Protocol):
    def notify(self, message: str) -> None: ...

def alert_all(subscribers: list[Notifiable], message: str) -> None:
    for s in subscribers:
        s.notify(message)   # works for ANY object with a .notify(), no inheritance needed
```
Mention out loud when you use this over `ABC`: "I'm using a Protocol here since I just need structural compatibility, not a real is-a relationship or shared implementation."

## 3. `@dataclass` — the idiomatic value object

Equivalent role to Java records. Auto-generates `__init__`, `__repr__`, `__eq__`. Use `frozen=True` for immutability (equivalent to Java's `final` fields).

```python
from dataclasses import dataclass, field

@dataclass(frozen=True)
class Money:
    cents: int

    def add(self, other: "Money") -> "Money":
        return Money(self.cents + other.cents)     # returns new instance — respects frozen immutability

@dataclass
class ParkingTicket:
    vehicle: "Vehicle"
    entry_time: "datetime"
    exit_time: "datetime | None" = None            # default value
    events: list[str] = field(default_factory=list) # mutable default MUST use default_factory, never `= []`
```
**Interview trap:** `events: list = []` as a default is a classic Python bug (shared mutable default across all instances) — always use `field(default_factory=list)`. Bringing this up unprompted is a strong signal.

## 4. Properties — not manual getters/setters

Writing `get_x()`/`set_x()` everywhere is the "Java in Python syntax" anti-pattern the evaluation framework warns about. Use plain attributes by default; add `@property` only when you need validation or computed values.

```python
class ParkingSpot:
    def __init__(self, spot_id: str, size: "VehicleSize"):
        self._spot_id = spot_id
        self.size = size
        self._parked_vehicle: "Vehicle | None" = None

    @property
    def is_free(self) -> bool:                 # read as spot.is_free, no parens — computed property
        return self._parked_vehicle is None

    @property
    def parked_vehicle(self) -> "Vehicle | None":
        return self._parked_vehicle

    @parked_vehicle.setter
    def parked_vehicle(self, vehicle: "Vehicle") -> None:
        if self._parked_vehicle is not None:
            raise ValueError("Spot already occupied")   # validation on assignment
        self._parked_vehicle = vehicle
```

## 5. Enums — with behavior, like Java

```python
from enum import Enum, auto

class VehicleSize(Enum):
    MOTORCYCLE = 1
    CAR = 2
    TRUCK = 4

    @property
    def spots_required(self) -> int:
        return self.value

class Direction(Enum):
    UP = auto()
    DOWN = auto()

    def opposite(self) -> "Direction":
        return Direction.DOWN if self is Direction.UP else Direction.UP
```

## 6. Multiple inheritance & MRO — Python allows what Java doesn't

Python supports multiple inheritance of classes (Java only allows multiple *interfaces*). Method Resolution Order (MRO, via C3 linearization) decides which parent's method wins — inspect with `ClassName.__mro__`. In LLD interviews, prefer **composition or mixins** over deep multiple inheritance; a "mixin" (small class adding one capability, not meant to stand alone) is the Pythonic equivalent of a Java default-method interface.

```python
class TimestampMixin:
    def touch(self):
        self.updated_at = datetime.now()

class LoggingMixin:
    def log(self, msg):
        print(f"[{self.__class__.__name__}] {msg}")

class Ticket(TimestampMixin, LoggingMixin):
    ...

print(Ticket.__mro__)  # shows resolution order left-to-right, depth-first-ish (C3)
```

## 7. Dunder methods — Python's answer to `equals`/`toString`/`Comparable`

| Java | Python equivalent |
|---|---|
| `equals()` | `__eq__` |
| `hashCode()` | `__hash__` |
| `toString()` | `__str__` / `__repr__` |
| `compareTo()` (Comparable) | `__lt__` (+ `functools.total_ordering`, or just implement all of `__lt__/__le__/__eq__/...`) |
| operator overload (`+`) | `__add__` |

```python
from functools import total_ordering

@total_ordering
class ParkingTicket:
    def __init__(self, entry_time):
        self.entry_time = entry_time

    def __eq__(self, other):
        return self.entry_time == other.entry_time
    def __lt__(self, other):
        return self.entry_time < other.entry_time   # total_ordering derives <=, >, >= from this + __eq__
    def __repr__(self):
        return f"ParkingTicket(entry_time={self.entry_time})"
```
`@dataclass` gives you `__eq__`/`__repr__` for free — only hand-write dunders when the class isn't a plain dataclass or needs custom equality logic.

## 8. `Comparator`-equivalent: pass a `key=` function, don't build a class

Java needs a `Comparator<T>` object; Python idiom is just a callable passed as `key=`:
```python
tickets.sort(key=lambda t: t.entry_time)                    # ascending by entry_time
tickets.sort(key=lambda t: t.fee, reverse=True)              # descending by fee

import heapq
heapq.heapify(requests)   # requests' elements need __lt__, or push (priority, item) tuples
```

## 9. Standard-library structures you should reach for automatically

| Need | Use |
|---|---|
| Insertion-ordered map, O(1) move-to-end | `collections.OrderedDict` (or plain `dict`, ordered since 3.7 — but `OrderedDict.move_to_end()` is what makes **LRU cache** trivial) |
| Counting | `collections.Counter` |
| Default-on-missing-key | `collections.defaultdict` |
| Double-ended queue (stack+queue) | `collections.deque` |
| Priority queue | `heapq` (push `(priority, item)` tuples) |
| Immutable lightweight record | `collections.namedtuple` or `@dataclass(frozen=True)` |

```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self._store: OrderedDict = OrderedDict()

    def get(self, key):
        if key not in self._store: return -1
        self._store.move_to_end(key)     # mark as recently used
        return self._store[key]

    def put(self, key, value):
        if key in self._store:
            self._store.move_to_end(key)
        self._store[key] = value
        if len(self._store) > self.capacity:
            self._store.popitem(last=False)   # evict least-recently-used (front)
```

## 10. Type hints — expected in a "senior Python" LLD answer

Always type-hint public method signatures; it's the closest Python gets to Java's compile-time contract-visibility and interviewers read it as rigor.

```python
def find_available_spot(self, vehicle: "Vehicle") -> "ParkingSpot | None":
    ...
```

## 11. Context managers — for anything acquire/release shaped

If a problem has an acquire/release or lock/unlock shape (e.g., reserving a seat, acquiring a resource), a context manager is the idiomatic Python answer to Java's try/finally or try-with-resources:

```python
from contextlib import contextmanager

@contextmanager
def reserve_seat(seat: "Seat"):
    seat.lock()
    try:
        yield seat
    finally:
        seat.unlock()          # guaranteed release, mirrors Java try-with-resources / try-finally

with reserve_seat(seat) as s:
    process_payment(s)
```

## Continue

Next: [05-java-vs-python-cheatsheet.md](05-java-vs-python-cheatsheet.md)
