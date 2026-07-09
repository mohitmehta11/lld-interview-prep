# Notification and Observer System

## Requirements

- "What's a concrete example of a subject?" — Pick one to ground the design and state it: an `Order` whose status changes (`PLACED → SHIPPED → DELIVERED`); subscribers are users who want updates on that specific order. **You decide** if the interviewer wants something more abstract — the mechanism is identical for a stock-price ticker or a topic/channel subscription.
- "Push or pull model?" — Assume push: the subject sends the new state directly in the notification payload. **You decide** to mention pull (subject only sends "something changed," observer calls back to fetch details) as the alternative — useful when the payload is expensive to compute and not every observer needs it.
- "One channel or many, and does the user pick?" — Assume a user can subscribe with a preferred channel (email/SMS/push) per subscription; the *channel send mechanism* is a Strategy behind a common interface, decoupled from the *subscription/notify* mechanism, which is Observer. This is the crux of the problem: two different patterns solving two different variabilities.
- "Sync or async dispatch?" — Design synchronous first for correctness and clarity; call out async dispatch (thread pool fan-out) as the answer to "what if there are 10,000 observers," referencing [../06-concurrency-essentials.md](../06-concurrency-essentials.md).
- "Is this the same thing as an event bus / message queue?" — No, and this distinction is worth stating explicitly to the interviewer: **Observer** = the subject holds direct references to its observers and calls them synchronously/in-process; **event bus / pub-sub broker** = a decoupled intermediary (topic, message queue) sits between publishers and subscribers, neither knows about the other, and delivery is typically async/out-of-process. This file implements Observer; an event bus would replace the `Subject`'s direct observer list with "publish to broker" + "broker fans out to registered subscribers," and is the natural next step if you're asked to decouple further.

**In scope**: `Subject`/`Observer` interfaces (own implementation, not `java.util.Observer`), a concrete `Order` subject, per-subscription channel preference via `NotificationChannel` strategy (Email/SMS/Push), subscribe/unsubscribe including the memory-leak discussion, sync dispatch with a note on async.

**Out of scope**: message persistence/durability, exactly-once delivery guarantees, an actual broker/queue implementation (mentioned as the follow-up direction, not built).

## Core entities & relationships

```
Subject (interface)
  └─ has-a[*] Observer (interface)      — subject holds DIRECT references; this is what distinguishes Observer from an event bus

Order (implements Subject)
  └─ has-a[1] OrderStatus enum

Subscription (implements Observer)
  ├─ has-a[1] User
  └─ has-a[1] NotificationChannel (interface)   — Strategy: how this observer wants to be reached
        ├─ EmailChannel
        ├─ SmsChannel
        └─ PushChannel
              └─ decorated by[*] ChannelDecorator (interface, extends NotificationChannel)
                    └─ ThrottlingChannel        — batches/rate-limits sends, optional
```

**Why `Subscription` (a User + a channel choice) is the `Observer`, not `User` directly**: a single user may want order-status updates via push but security-alert updates via email — the observer identity that a `Subject` notifies needs to carry *which channel this particular subscription uses*, which a bare `User` reference can't express without the subject reaching into user preferences itself (a DIP violation — the subject shouldn't need to know how preferences are stored). Wrapping `User` + `NotificationChannel` in a `Subscription` keeps `Order` fully decoupled from that policy.

**Why channel selection is Strategy, layered *underneath* Observer, not folded into it**: "how do I notify one observer" (Strategy: pick email vs SMS vs push) and "who gets notified and when" (Observer: subject's list, triggered on state change) are two independent axes of change. A new channel (WhatsApp) shouldn't touch `Order` or the notify loop; a new subject (add a `StockTicker`) shouldn't touch channel logic. Keeping them as two separate interfaces is what makes both axes independently extensible.

## Design patterns applied

- [Observer](../patterns/03-behavioral-patterns.md#observer) — the centerpiece of this file: `Order` (subject) maintains a growable, runtime-mutable list of `Subscription` (observer) instances and pushes state changes to all of them without knowing what they are or how many there are; this is the pattern's textbook shape and this file implements it from scratch rather than via `java.util.Observer`/`Observable`, which were **deprecated in Java 9** (no generics, forces subclassing `Observable` instead of an interface, no delivery-order or thread-safety guarantees) — every real Java codebase rolls its own `Subject`/`Observer` pair or uses a library (e.g. Guava `EventBus`, Spring's `ApplicationEventPublisher`), exactly as done below.
- [Strategy](../patterns/03-behavioral-patterns.md#strategy) — `NotificationChannel` (Email/SMS/Push) varies *how* a single notification is delivered, completely orthogonal to *who* gets notified; new channels are new classes, zero changes to `Order` or the subscription list.
- [Decorator](../patterns/02-structural-patterns.md#decorator) *(optional)* — `ThrottlingChannel` wraps a base channel to batch or rate-limit sends (e.g. collapse 10 stock-price ticks into 1 email per minute) without the base `EmailChannel` needing to know about batching — additive behavior, not a new channel type.

## Java implementation

```java
import java.util.*;
import java.util.concurrent.*;

enum OrderStatus { PLACED, SHIPPED, DELIVERED, CANCELLED }

// Rolled ourselves rather than extending java.util.Observable (deprecated since Java 9,
// no generics, forces inheritance). Push model: payload travels with the event.
interface Observer<T> {
    void onUpdate(T event);
}

interface Subject<T> {
    void subscribe(Observer<T> observer);
    void unsubscribe(Observer<T> observer);
    void notifyObservers(T event);
}

final class OrderStatusChanged {
    final String orderId;
    final OrderStatus newStatus;
    OrderStatusChanged(String orderId, OrderStatus newStatus) {
        this.orderId = orderId; this.newStatus = newStatus;
    }
}

final class Order implements Subject<OrderStatusChanged> {
    private final String orderId;
    private OrderStatus status = OrderStatus.PLACED;
    // CopyOnWriteArrayList: safe to iterate-and-notify while another thread subscribes/unsubscribes
    // concurrently — see ../06-concurrency-essentials.md for the trade-off vs a synchronized ArrayList.
    private final List<Observer<OrderStatusChanged>> observers = new CopyOnWriteArrayList<>();

    Order(String orderId) { this.orderId = orderId; }

    public void subscribe(Observer<OrderStatusChanged> observer) { observers.add(observer); }

    // Without this, every Subscription ever created stays reachable via Order's list forever —
    // the canonical Observer memory leak. Callers MUST unsubscribe on teardown (see walkthrough).
    public void unsubscribe(Observer<OrderStatusChanged> observer) { observers.remove(observer); }

    public void notifyObservers(OrderStatusChanged event) {
        for (Observer<OrderStatusChanged> o : observers) o.onUpdate(event);
    }

    void updateStatus(OrderStatus newStatus) {
        this.status = newStatus;
        notifyObservers(new OrderStatusChanged(orderId, newStatus));
    }
}

// --- Strategy: delivery mechanism, orthogonal to subscription bookkeeping ---
interface NotificationChannel {
    void send(String recipient, String message);
}

final class EmailChannel implements NotificationChannel {
    public void send(String recipient, String message) {
        System.out.println("EMAIL to " + recipient + ": " + message);
    }
}

final class SmsChannel implements NotificationChannel {
    public void send(String recipient, String message) {
        System.out.println("SMS to " + recipient + ": " + message);
    }
}

final class PushChannel implements NotificationChannel {
    public void send(String recipient, String message) {
        System.out.println("PUSH to " + recipient + ": " + message);
    }
}

// --- Observer implementation: a User + their chosen channel for THIS subscription ---
final class Subscription implements Observer<OrderStatusChanged> {
    private final String userContact;
    private final NotificationChannel channel;

    Subscription(String userContact, NotificationChannel channel) {
        this.userContact = userContact; this.channel = channel;
    }

    public void onUpdate(OrderStatusChanged event) {
        String message = "Order " + event.orderId + " is now " + event.newStatus;
        try {
            channel.send(userContact, message);
        } catch (RuntimeException sendFailure) {
            // A single channel outage must not break notifyObservers()'s loop over other observers.
            System.err.println("Delivery failed for " + userContact + ": " + sendFailure.getMessage());
        }
    }
}

// --- Async dispatch for the "thousands of observers" follow-up ---
final class AsyncOrder implements Subject<OrderStatusChanged> {
    private final Order delegate;
    private final ExecutorService pool;

    AsyncOrder(Order delegate, ExecutorService pool) { this.delegate = delegate; this.pool = pool; }

    public void subscribe(Observer<OrderStatusChanged> o) { delegate.subscribe(o); }
    public void unsubscribe(Observer<OrderStatusChanged> o) { delegate.unsubscribe(o); }

    public void notifyObservers(OrderStatusChanged event) {
        // Fan out onto the pool instead of the calling thread; one slow observer can't stall the rest.
        pool.submit(() -> delegate.notifyObservers(event));
    }
}
```

## Python implementation

```python
from __future__ import annotations
from abc import ABC, abstractmethod
from concurrent.futures import ThreadPoolExecutor
from dataclasses import dataclass
from enum import Enum, auto
from typing import Generic, TypeVar
import weakref

T = TypeVar("T")


class OrderStatus(Enum):
    PLACED = auto()
    SHIPPED = auto()
    DELIVERED = auto()
    CANCELLED = auto()


class Observer(ABC, Generic[T]):
    @abstractmethod
    def on_update(self, event: T) -> None: ...


class Subject(ABC, Generic[T]):
    @abstractmethod
    def subscribe(self, observer: Observer[T]) -> None: ...
    @abstractmethod
    def unsubscribe(self, observer: Observer[T]) -> None: ...
    @abstractmethod
    def notify_observers(self, event: T) -> None: ...


@dataclass(frozen=True)
class OrderStatusChanged:
    order_id: str
    new_status: OrderStatus


class Order(Subject[OrderStatusChanged]):
    def __init__(self, order_id: str):
        self.order_id = order_id
        self.status = OrderStatus.PLACED
        # WeakSet avoids the classic Observer leak: an observer that's otherwise garbage
        # can be collected without an explicit unsubscribe(); explicit unsubscribe is still
        # the correct primary mechanism — this is a safety net, not a substitute.
        self._observers: "weakref.WeakSet[Observer[OrderStatusChanged]]" = weakref.WeakSet()

    def subscribe(self, observer: Observer[OrderStatusChanged]) -> None:
        self._observers.add(observer)

    def unsubscribe(self, observer: Observer[OrderStatusChanged]) -> None:
        self._observers.discard(observer)

    def notify_observers(self, event: OrderStatusChanged) -> None:
        for observer in list(self._observers):  # snapshot: safe against mutation during iteration
            observer.on_update(event)

    def update_status(self, new_status: OrderStatus) -> None:
        self.status = new_status
        self.notify_observers(OrderStatusChanged(self.order_id, new_status))


# --- Strategy: delivery mechanism ---
class NotificationChannel(ABC):
    @abstractmethod
    def send(self, recipient: str, message: str) -> None: ...


class EmailChannel(NotificationChannel):
    def send(self, recipient: str, message: str) -> None:
        print(f"EMAIL to {recipient}: {message}")


class SmsChannel(NotificationChannel):
    def send(self, recipient: str, message: str) -> None:
        print(f"SMS to {recipient}: {message}")


class PushChannel(NotificationChannel):
    def send(self, recipient: str, message: str) -> None:
        print(f"PUSH to {recipient}: {message}")


class Subscription(Observer[OrderStatusChanged]):
    def __init__(self, user_contact: str, channel: NotificationChannel):
        self.user_contact = user_contact
        self.channel = channel

    def on_update(self, event: OrderStatusChanged) -> None:
        message = f"Order {event.order_id} is now {event.new_status.name}"
        try:
            self.channel.send(self.user_contact, message)
        except Exception as send_failure:  # one bad channel must not break the fan-out loop
            print(f"Delivery failed for {self.user_contact}: {send_failure}")


# --- Async dispatch for the "thousands of observers" follow-up ---
class AsyncOrder(Subject[OrderStatusChanged]):
    def __init__(self, delegate: Order, pool: ThreadPoolExecutor):
        self.delegate = delegate
        self.pool = pool

    def subscribe(self, observer: Observer[OrderStatusChanged]) -> None:
        self.delegate.subscribe(observer)

    def unsubscribe(self, observer: Observer[OrderStatusChanged]) -> None:
        self.delegate.unsubscribe(observer)

    def notify_observers(self, event: OrderStatusChanged) -> None:
        self.pool.submit(self.delegate.notify_observers, event)
```

## Sample walkthrough

```python
order = Order("ORD-1001")

alice_sub = Subscription("alice@example.com", EmailChannel())
bob_sub = Subscription("+1-555-0100", SmsChannel())

order.subscribe(alice_sub)
order.subscribe(bob_sub)

order.update_status(OrderStatus.SHIPPED)
# -> EMAIL to alice@example.com: Order ORD-1001 is now SHIPPED
# -> SMS to +1-555-0100: Order ORD-1001 is now SHIPPED

order.unsubscribe(bob_sub)          # bob no longer wants updates — explicit teardown, not just weakref hope
order.update_status(OrderStatus.DELIVERED)
# -> EMAIL to alice@example.com: Order ORD-1001 is now DELIVERED   (bob correctly silent)
```

## Follow-up questions

- **"A channel send fails intermittently (network blip on SMS) — what's the retry story?"** Wrap `NotificationChannel` in a `RetryingChannel` decorator with bounded retries + backoff, or push failed sends onto a dead-letter queue for later replay; `Subscription.on_update` already isolates one failure from breaking the fan-out loop, so retry logic slots in at the channel layer without touching `Order`.
- **"There are 50,000 observers on one hot subject — synchronous notify is too slow."** Switch to `AsyncOrder`/the pool-based dispatch shown above, referencing [../06-concurrency-essentials.md](../06-concurrency-essentials.md) for sizing the executor and deciding fire-and-forget vs waiting on `Future`s for delivery confirmation; note the trade-off that async means the caller (`update_status`) returns before delivery is confirmed.
- **"Users want to configure channel preference per event type, not per subscription."** Extend `Subscription` to hold a `Map<EventType, NotificationChannel>` (or one `Subscription` per event type, which is simpler and matches the existing model) — the `Order`/`Subject` side is untouched either way since it only ever calls `on_update`.
- **"Isn't this the same as an event bus — why not just use one?"** No: here `Order` holds direct references to its observers (tight but simple, in-process, synchronous-by-default). An event bus decouples further — publishers and subscribers never reference each other, a broker in between handles routing/delivery, usually async and often out-of-process (Kafka/SQS-style). Reach for an event bus when you have *many unrelated subject types* publishing to *many unrelated subscriber types* and don't want N×M direct wiring; Observer is the right minimal tool when one subject's state change matters to a bounded, directly-known set of listeners.
- **"How do we avoid the memory leak where forgotten subscriptions pile up?"** Discipline first: every `subscribe` should have a matching `unsubscribe` on teardown (e.g. when a user closes the order-tracking screen). As a safety net (not a substitute), the Python version uses `weakref.WeakSet` so a `Subscription` with no other live references is collectible even if `unsubscribe` was missed; the Java equivalent would need `WeakReference`-wrapped observers or a periodic sweep, since `CopyOnWriteArrayList` holds strong references — call this trade-off out explicitly if asked.

## Common mistakes on this problem

- Reaching for `java.util.Observer`/`Observable` in Java — deprecated since Java 9, untyped (`Object arg`), and forces the subject to *extend* `Observable` rather than implement an interface, which blocks it from extending anything else. Always roll a small generic `Subject`/`Observer` pair instead.
- Conflating Observer with pub-sub/event-bus when asked to compare them — the direct-reference-vs-broker distinction (see follow-ups above) is a common probe, and hand-waving "they're basically the same" reads as a shallow pattern vocabulary rather than understood trade-offs.
- Baking the channel choice (email vs SMS) directly into the `Observer.onUpdate` implementation as an `if/else` on a `channelType` field, instead of injecting a `NotificationChannel` strategy — collapses two independent axes of variability into one class and makes adding WhatsApp support require editing existing subscription code.
- Never addressing the unsubscribe path at all — a design that only shows `subscribe()` and a working notify loop but no teardown story is an easy "what happens after 10,000 users unsubscribe... did they, though?" follow-up that exposes an unbounded-growth bug.

## Continue

This is the last file in `problems/`. For closing out interview prep, read [../07-common-mistakes.md](../07-common-mistakes.md) (what actually loses points across all of these problems) followed by [../08-final-checklist.md](../08-final-checklist.md) (the pre-interview 10-minute drill).
