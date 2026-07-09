# LRU Cache & Rate Limiter

Two warm-up problems in one file on purpose: both are graded on axis 2 (right data structure) more than axis 4 (patterns). The recurring trap is building a class hierarchy where the interviewer wanted you to reach for the right structure from `../03-java-oop-essentials.md#9-collections-framework--the-part-youll-use-constantly` / `../04-python-oop-essentials.md#9-standard-library-structures-you-should-reach-for-automatically` in one line, then layer patterns only where real variability exists (rate-limiting algorithm choice).

## Requirements

### LRU Cache

- "What's the capacity model — fixed at construction, or resizable?" → **You decide**: fixed capacity passed to the constructor; resizing is an out-of-scope follow-up.
- "Do we need thread safety?" → **You decide**: assume single-threaded core; mention the lock wrapper as a follow-up (see Concurrency note below).
- "Should `get` count as a "use" for recency, or only `put`?" → **You decide**: both `get` and `put` refresh recency — that's the definition of LRU (vs LFU, which tracks frequency, not recency).
- In scope: `get(key) -> value`, `put(key, value)`, O(1) average time for both, eviction of the least-recently-used entry on overflow.
- Out of scope: LFU variant, TTL-based expiry (that's the rate limiter's concern below), persistence, distributed cache coherence.

### Rate Limiter

- "Per-user or global limiter?" → **You decide**: per-key (e.g., per user ID or API key) — a single global limiter is a degenerate case of the same design with one key.
- "Which algorithm?" → **You decide**: support token bucket and sliding-window log behind one interface; the interviewer usually wants to see you *swap* algorithms, not commit to one.
- "What happens on rejection — block, queue, or drop?" → **You decide**: drop (return `false`/deny) — queuing turns this into a different problem (a scheduler).
- In scope: `allowRequest(key) -> boolean`, pluggable algorithm, per-key isolation.
- Out of scope: distributed rate limiting (Redis-backed counters), leaky bucket and fixed-window full implementations (discussed, not coded), backpressure/queueing.

## Core entities & relationships

```
LRUCache
  ├─ has-a[1] Map<Key, Node>        (hash index for O(1) lookup)
  └─ has-a[1] doubly-linked list    (recency order; head=MRU, tail=LRU)

RateLimiter (interface)
  ├─ implemented-by TokenBucketRateLimiter
  └─ implemented-by SlidingWindowRateLimiter

RateLimiterRegistry
  └─ has-a[*] RateLimiter            (one instance per key, e.g. per user)
```

The LRU cache is deliberately *not* modeled as a class hierarchy — there's no polymorphic seam here, just a data-structure choice (hash map for O(1) lookup + linked list for O(1) reorder/evict). Forcing a Strategy or Template Method onto this is the single most common overuse mistake on this problem (see `../patterns/00-overview.md` overuse trap). The rate limiter is the opposite case: the algorithm *is* the variability, so it gets a real interface.

## Design patterns applied

- [Strategy](../patterns/03-behavioral-patterns.md#strategy) — `RateLimiter` is the interface; `TokenBucketRateLimiter` and `SlidingWindowRateLimiter` are interchangeable policies selected at construction, satisfying [OCP](../02-solid-principles.md#o--openclosed-principle): a new algorithm (leaky bucket) means a new class, zero edits to callers.
- No pattern on the LRU cache itself — noted explicitly because *not* applying one is the correct call here; the variability (eviction policy) is a single fixed rule, not something that swaps at runtime. If an interviewer asks for a pluggable eviction policy (LRU vs LFU vs FIFO), *then* Strategy becomes justified — see Follow-up questions.

## Java implementation

### LRU Cache — idiomatic one-liner-ish version

`LinkedHashMap` already maintains insertion/access order and gives you eviction for free via `removeEldestEntry`. Reach for this when the interviewer says "use the language" rather than "build it from scratch" — naming it explicitly ("I'm using `LinkedHashMap` in access-order mode, which is exactly an LRU list + hash index under the hood") shows library fluency without hiding the mechanism.

```java
import java.util.LinkedHashMap;
import java.util.Map;

public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;

    public LRUCache(int capacity) {
        // accessOrder=true: get() and put() both move an entry to the end
        super(capacity, 0.75f, true);
        this.capacity = capacity;
    }

    public V get(Object key) {
        return super.getOrDefault(key, null); // getOrDefault still triggers access-order reorder
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity; // called by put()/putAll() after insertion
    }
}
```

Caveat to say out loud: `LinkedHashMap` in access-order mode reorders on iteration too (`entrySet()`, `keySet()`), which can silently perturb LRU order if you're not careful — fine for this problem since we don't expose iteration.

### LRU Cache — manual doubly-linked-list + HashMap (from scratch)

Some interviewers explicitly forbid `LinkedHashMap` to test whether you actually understand the mechanism. This is the fallback — a `HashMap<K, Node>` for O(1) lookup plus a hand-rolled doubly-linked list (sentinel head/tail nodes to avoid null-checks) for O(1) move-to-front/evict.

```java
import java.util.HashMap;
import java.util.Map;

public class LRUCacheManual<K, V> {
    private static class Node<K, V> {
        K key; V value;
        Node<K, V> prev, next;
        Node(K k, V v) { key = k; value = v; }
    }

    private final int capacity;
    private final Map<K, Node<K, V>> index = new HashMap<>();
    private final Node<K, V> head = new Node<>(null, null); // sentinel, head.next = MRU
    private final Node<K, V> tail = new Node<>(null, null); // sentinel, tail.prev = LRU

    public LRUCacheManual(int capacity) {
        this.capacity = capacity;
        head.next = tail;
        tail.prev = head;
    }

    public V get(K key) {
        Node<K, V> node = index.get(key);
        if (node == null) return null;
        moveToFront(node);
        return node.value;
    }

    public void put(K key, V value) {
        Node<K, V> existing = index.get(key);
        if (existing != null) {
            existing.value = value;
            moveToFront(existing);
            return;
        }
        if (index.size() == capacity) {
            Node<K, V> lru = tail.prev;
            remove(lru);
            index.remove(lru.key);
        }
        Node<K, V> node = new Node<>(key, value);
        index.put(key, node);
        insertAfterHead(node);
    }

    private void moveToFront(Node<K, V> node) {
        remove(node);
        insertAfterHead(node);
    }

    private void remove(Node<K, V> node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }

    private void insertAfterHead(Node<K, V> node) {
        node.next = head.next;
        node.prev = head;
        head.next.prev = node;
        head.next = node;
    }
}
```

### Rate Limiter — Strategy

```java
public interface RateLimiter {
    boolean allowRequest(long nowMillis);
}

/** Refills tokens continuously; allows bursts up to bucket capacity. */
public class TokenBucketRateLimiter implements RateLimiter {
    private final long capacity;
    private final double refillPerMillis; // tokens added per millisecond
    private double tokens;
    private long lastRefillTime;

    public TokenBucketRateLimiter(long capacity, double refillPerSecond, long nowMillis) {
        this.capacity = capacity;
        this.refillPerMillis = refillPerSecond / 1000.0;
        this.tokens = capacity;
        this.lastRefillTime = nowMillis;
    }

    @Override
    public synchronized boolean allowRequest(long nowMillis) {
        refill(nowMillis);
        if (tokens >= 1.0) {
            tokens -= 1.0;
            return true;
        }
        return false;
    }

    private void refill(long nowMillis) {
        long elapsed = nowMillis - lastRefillTime;
        if (elapsed <= 0) return;
        tokens = Math.min(capacity, tokens + elapsed * refillPerMillis);
        lastRefillTime = nowMillis;
    }
}

/** Exact rate over a rolling window; stricter than token bucket, no burst tolerance. */
public class SlidingWindowRateLimiter implements RateLimiter {
    private final int maxRequests;
    private final long windowMillis;
    private final java.util.Deque<Long> timestamps = new java.util.ArrayDeque<>();

    public SlidingWindowRateLimiter(int maxRequests, long windowMillis) {
        this.maxRequests = maxRequests;
        this.windowMillis = windowMillis;
    }

    @Override
    public synchronized boolean allowRequest(long nowMillis) {
        while (!timestamps.isEmpty() && nowMillis - timestamps.peekFirst() >= windowMillis) {
            timestamps.pollFirst(); // drop entries that aged out of the window
        }
        if (timestamps.size() < maxRequests) {
            timestamps.addLast(nowMillis);
            return true;
        }
        return false;
    }
}

/** Per-key isolation; one RateLimiter instance per key, lazily created. */
public class RateLimiterRegistry {
    private final java.util.Map<String, RateLimiter> limiters = new java.util.concurrent.ConcurrentHashMap<>();
    private final java.util.function.Supplier<RateLimiter> factory;

    public RateLimiterRegistry(java.util.function.Supplier<RateLimiter> factory) {
        this.factory = factory;
    }

    public boolean allow(String key, long nowMillis) {
        return limiters.computeIfAbsent(key, k -> factory.get()).allowRequest(nowMillis);
    }
}
```

Fixed-window counters (reset a counter every N seconds — simple but allows 2x burst at window boundaries) and leaky bucket (fixed-rate outflow, smooths bursts more aggressively than token bucket) are the other two textbook algorithms; both slot in as one more `RateLimiter` implementation each, no other code changes — that's the OCP payoff of the Strategy seam.

## Python implementation

```python
from collections import OrderedDict
from typing import Optional, Any

class LRUCache:
    """Idiomatic version: OrderedDict.move_to_end + popitem(last=False)."""
    def __init__(self, capacity: int):
        self.capacity = capacity
        self._store: OrderedDict[Any, Any] = OrderedDict()

    def get(self, key) -> Optional[Any]:
        if key not in self._store:
            return None
        self._store.move_to_end(key)
        return self._store[key]

    def put(self, key, value) -> None:
        if key in self._store:
            self._store.move_to_end(key)
        self._store[key] = value
        if len(self._store) > self.capacity:
            self._store.popitem(last=False)  # evict LRU (front of the dict)


class _Node:
    __slots__ = ("key", "value", "prev", "next")
    def __init__(self, key=None, value=None):
        self.key, self.value, self.prev, self.next = key, value, None, None


class LRUCacheManual:
    """From-scratch version: dict for O(1) lookup + hand-rolled doubly-linked list."""
    def __init__(self, capacity: int):
        self.capacity = capacity
        self._index: dict[Any, _Node] = {}
        self._head, self._tail = _Node(), _Node()   # sentinels
        self._head.next, self._tail.prev = self._tail, self._head

    def get(self, key) -> Optional[Any]:
        node = self._index.get(key)
        if node is None:
            return None
        self._move_to_front(node)
        return node.value

    def put(self, key, value) -> None:
        node = self._index.get(key)
        if node is not None:
            node.value = value
            self._move_to_front(node)
            return
        if len(self._index) == self.capacity:
            lru = self._tail.prev
            self._remove(lru)
            del self._index[lru.key]
        node = _Node(key, value)
        self._index[key] = node
        self._insert_after_head(node)

    def _move_to_front(self, node: _Node) -> None:
        self._remove(node)
        self._insert_after_head(node)

    def _remove(self, node: _Node) -> None:
        node.prev.next, node.next.prev = node.next, node.prev

    def _insert_after_head(self, node: _Node) -> None:
        node.next, node.prev = self._head.next, self._head
        self._head.next.prev, self._head.next = node, node
```

```python
from abc import ABC, abstractmethod
from collections import deque
import threading
import time


class RateLimiter(ABC):
    @abstractmethod
    def allow_request(self, now: float) -> bool: ...


class TokenBucketRateLimiter(RateLimiter):
    """Continuous refill; allows bursts up to bucket capacity."""
    def __init__(self, capacity: float, refill_per_second: float, now: float):
        self.capacity = capacity
        self.refill_per_second = refill_per_second
        self.tokens = capacity
        self.last_refill = now
        self._lock = threading.Lock()

    def allow_request(self, now: float) -> bool:
        with self._lock:
            elapsed = now - self.last_refill
            if elapsed > 0:
                self.tokens = min(self.capacity, self.tokens + elapsed * self.refill_per_second)
                self.last_refill = now
            if self.tokens >= 1:
                self.tokens -= 1
                return True
            return False


class SlidingWindowRateLimiter(RateLimiter):
    """Exact rolling-window count; no burst tolerance."""
    def __init__(self, max_requests: int, window_seconds: float):
        self.max_requests = max_requests
        self.window_seconds = window_seconds
        self.timestamps: deque[float] = deque()
        self._lock = threading.Lock()

    def allow_request(self, now: float) -> bool:
        with self._lock:
            while self.timestamps and now - self.timestamps[0] >= self.window_seconds:
                self.timestamps.popleft()
            if len(self.timestamps) < self.max_requests:
                self.timestamps.append(now)
                return True
            return False


class RateLimiterRegistry:
    """Per-key isolation; one limiter instance per key, lazily created."""
    def __init__(self, factory):
        self._factory = factory
        self._limiters: dict[str, RateLimiter] = {}
        self._lock = threading.Lock()

    def allow(self, key: str, now: float) -> bool:
        with self._lock:
            limiter = self._limiters.setdefault(key, self._factory())
        return limiter.allow_request(now)
```

## Sample walkthrough

```python
cache = LRUCache(capacity=2)
cache.put("a", 1)
cache.put("b", 2)
cache.get("a")          # "a" now most-recently-used; order is b, a
cache.put("c", 3)       # capacity exceeded -> evicts "b" (the LRU)
assert cache.get("b") is None
assert cache.get("a") == 1
assert cache.get("c") == 3

registry = RateLimiterRegistry(lambda: TokenBucketRateLimiter(capacity=5, refill_per_second=1, now=time.time()))
now = time.time()
for _ in range(5):
    assert registry.allow("user-42", now) is True   # burst of 5 consumes the full bucket
assert registry.allow("user-42", now) is False       # 6th request in the same instant is denied
assert registry.allow("user-99", now) is True        # different key, isolated bucket
```

## Follow-up questions

- **"Make the eviction policy pluggable (LRU vs LFU vs FIFO)."** This is where Strategy *does* become justified on the cache: extract an `EvictionPolicy` interface with `onAccess(key)`/`onInsert(key)`/`evictionCandidate()`, and have `LRUCache` delegate to it instead of hardcoding recency order.
- **"Support TTL expiry per entry."** Store `(value, expiresAt)` in the node/value slot; check-and-evict lazily on `get`, plus an optional background sweep — doesn't change the core structure.
- **"Make the cache thread-safe under concurrent readers/writers."** Wrap the whole structure in one `ReentrantLock`/`synchronized` block (coarse-grained is fine here — LRU reordering on every `get` makes fine-grained per-node locking correctness-hazardous); see `../06-concurrency-essentials.md` for the primitives and trade-offs.
- **"Rate limiter needs to work across multiple app servers, not just in-process."** Swap the in-memory `Map`/`dict` in `RateLimiterRegistry` for a Redis-backed counter (`INCR` + `EXPIRE` for fixed-window, a Lua script for atomic token-bucket) — the `RateLimiter` interface doesn't change, only the implementation's storage.
- **"What if two algorithms need to run together (e.g., both a burst limit and a sustained-rate limit)?"** Compose two `RateLimiter` instances and require both to return `true` — a `CompositeRateLimiter` implementing the same interface, no client-code change.

## Common mistakes on this problem

- Building an `EvictionStrategy` interface with only one implementation for the cache — this is the overuse trap (see `../patterns/00-overview.md`): no interviewer-stated variability means no pattern yet.
- Using `ArrayList`/`list` for the LRU order and doing O(n) linear scans to find/move an entry — defeats the entire point of the problem; the interviewer is specifically listening for "hash map + doubly-linked list."
- Forgetting that `get` must also update recency (common bug: only `put` updates order, silently turning it into FIFO-with-a-cache-shaped-API).
- Hardcoding one rate-limiting algorithm inline in the endpoint/controller instead of behind an interface, then having no answer when asked "how would you A/B test two algorithms."

## Continue

Next: [06-splitwise-expense-sharing.md](06-splitwise-expense-sharing.md)
