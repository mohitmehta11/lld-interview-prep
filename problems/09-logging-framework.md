# Logging Framework

## Requirements

- "What log levels do we need, and do they nest?" — Assume the standard totally-ordered set `DEBUG < INFO < WARN < ERROR`; a logger configured at `INFO` suppresses `DEBUG` but emits `INFO` and above. **You decide** if not specified.
- "How many output destinations, and can a log line go to more than one?" — Assume yes: a single log call can fan out to console + file + network appenders simultaneously, each with its own independent level filter (e.g. file appender logs `DEBUG+`, network appender only `ERROR+`).
- "Sync or async?" — Design synchronous first (simpler, correct), call out async (queue + worker thread/pool) as a follow-up so it's clear you know the failure modes (backpressure, dropped logs, ordering) rather than defaulting to it and hand-waving.
- "Is this a library other teams consume, or an app-internal component?" — Treat it as a library: the public surface is a `Logger` built via a `LoggerBuilder`, obtained through a `LoggerFactory`, not a class apps subclass. This is a systems/infra design problem, not a UI-adjacent one — frame it that way in the interview (think `log4j`/`slf4j`/Python's `logging`, not a chat app's message list).
- "Do we need log rotation / structured (JSON) output?" — Out of scope for the core design; flagged as follow-ups since they're additive to `Appender` and don't change the core pipeline.

**In scope**: `Logger` with a configured minimum level, multiple pluggable `Appender`s (console/file/network stub), a level-based dispatch pipeline (Chain of Responsibility), fluent configuration (Builder), a process-wide registry (Singleton, justified), optional decorators for cross-cutting appender behavior (timestamp formatting, async flush).

**Out of scope**: distributed log aggregation (ELK/Splunk shipping), log rotation policy implementation detail (mentioned, not built), structured/JSON serialization format, sampling.

## Core entities & relationships

```
LoggerFactory (Singleton)
  └─ has-a[*] Logger                    (keyed by name, e.g. per-class/module)

Logger
  ├─ has-a[1] LogLevel                  (minimum level this logger accepts)
  ├─ has-a[*] Appender (interface)      (fan-out destinations)
  └─ built by[1] LoggerBuilder          (fluent construction)

Appender (interface)
  ├─ ConsoleAppender
  ├─ FileAppender
  ├─ NetworkAppender
  └─ decorated by[*] AppenderDecorator (interface, extends Appender)
        ├─ TimestampingAppender
        └─ AsyncAppender

LogLevelHandler (interface)             — Chain of Responsibility link
  DEBUG -> INFO -> WARN -> ERROR        (each decides "mine to handle, and/or pass down")
```

**Why `Logger` holds a *list* of `Appender`s rather than exactly one**: the requirement is explicit fan-out (console + file + network from one log call), and each `Appender` needs its *own* level filter independent of the `Logger`'s own level — a `Logger` set to `DEBUG` might still have a `NetworkAppender` that only forwards `ERROR+` to avoid flooding a remote sink. Two independent level checks (logger-level, then per-appender-level) is the key nuance graders look for.

**Why level dispatch is Chain of Responsibility and not a single `if (level >= threshold)` check**: that single check *is* a valid minimal implementation and is fine to state as the baseline — CoR earns its keep the moment you need level-specific side effects beyond filtering (e.g. `ERROR` additionally pages on-call, `WARN` additionally increments a metrics counter). Framing it as a chain from the start means adding that later is a new handler, not new branches in existing code. This file walks the chain explicitly since it's this knowledge base's canonical CoR example.

## Design patterns applied

- [Chain of Responsibility](../patterns/03-behavioral-patterns.md#chain-of-responsibility) — each `LogLevelHandler` (DEBUG→INFO→WARN→ERROR) decides whether it's responsible for a message and whether to pass it further down; a new level or level-specific side effect (e.g. ERROR also alerts) is a new link, not an edit to existing handlers. This is the textbook use case for the pattern in this knowledge base.
- [Builder](../patterns/01-creational-patterns.md#builder) — `LoggerBuilder` configures level + appenders + format, all optional/combinable knobs with sane defaults; a telescoping constructor (`Logger(level, appenders, format, async, ...)`) would be unreadable at 4+ knobs.
- [Singleton](../patterns/01-creational-patterns.md#singleton) — `LoggerFactory` is a genuinely justified Singleton: there is exactly one process-wide logger registry, and multiple instances would mean multiple independent configurations for "the same" named logger, which is a real bug, not just convenient sharing. Contrast with the overuse trap noted in [../patterns/01-creational-patterns.md](../patterns/01-creational-patterns.md#singleton) — this is the case where the smell ("need exactly one shared coordinator") is actually present in the requirements.
- [Decorator](../patterns/02-structural-patterns.md#decorator) — `TimestampingAppender`/`AsyncAppender` wrap a base `Appender` to add genuinely additive behavior (prefix a timestamp, offload to a background thread) without subclassing `FileAppender`, `ConsoleAppender`, and `NetworkAppender` three separate times each for "with timestamp" and "with async" — avoids the `N × M` subclass explosion.

## Java implementation

```java
import java.util.*;
import java.util.concurrent.*;
import java.time.Instant;

enum LogLevel { DEBUG, INFO, WARN, ERROR }

final class LogRecord {
    final LogLevel level;
    final String message;
    final Instant timestamp;
    LogRecord(LogLevel level, String message) {
        this.level = level; this.message = message; this.timestamp = Instant.now();
    }
}

interface Appender {
    void append(LogRecord record);
}

final class ConsoleAppender implements Appender {
    public void append(LogRecord r) { System.out.println("[" + r.level + "] " + r.message); }
}

final class FileAppender implements Appender {
    private final String path;
    FileAppender(String path) { this.path = path; }
    public void append(LogRecord r) {
        // Real impl: buffered writer + rotation hook; elided — see follow-ups for rotation.
        System.out.println("(file:" + path + ") [" + r.level + "] " + r.message);
    }
}

final class NetworkAppender implements Appender {
    private final String endpoint;
    NetworkAppender(String endpoint) { this.endpoint = endpoint; }
    public void append(LogRecord r) {
        System.out.println("(POST " + endpoint + ") [" + r.level + "] " + r.message);
    }
}

// --- Decorator: additive cross-cutting behavior, base Appender unmodified ---
abstract class AppenderDecorator implements Appender {
    protected final Appender delegate;
    AppenderDecorator(Appender delegate) { this.delegate = delegate; }
}

final class TimestampingAppender extends AppenderDecorator {
    TimestampingAppender(Appender delegate) { super(delegate); }
    public void append(LogRecord r) {
        // Mutate presentation only; delegate still owns the actual sink write.
        delegate.append(new LogRecord(r.level, "[" + r.timestamp + "] " + r.message));
    }
}

final class AsyncAppender extends AppenderDecorator {
    private final ExecutorService pool = Executors.newSingleThreadExecutor();
    AsyncAppender(Appender delegate) { super(delegate); }
    public void append(LogRecord r) {
        // Non-blocking from the caller's perspective; ordering preserved by single worker thread.
        // Unbounded submission here can OOM under sustained overload — see follow-ups.
        pool.submit(() -> delegate.append(r));
    }
}

// --- Chain of Responsibility: level-specific handling beyond a simple threshold check ---
abstract class LogLevelHandler {
    protected LogLevelHandler next;
    private final LogLevel handles;

    LogLevelHandler(LogLevel handles) { this.handles = handles; }
    LogLevelHandler setNext(LogLevelHandler next) { this.next = next; return next; }

    final void handle(LogRecord record, List<Appender> appenders) {
        if (record.level == handles) doHandle(record, appenders);
        if (next != null) next.handle(record, appenders); // levels are cumulative: ERROR also flows through WARN's side effects if wired that way
    }

    abstract void doHandle(LogRecord record, List<Appender> appenders);
}

final class DebugHandler extends LogLevelHandler {
    DebugHandler() { super(LogLevel.DEBUG); }
    void doHandle(LogRecord r, List<Appender> appenders) { dispatch(r, appenders); }
    private void dispatch(LogRecord r, List<Appender> appenders) { appenders.forEach(a -> a.append(r)); }
}

final class ErrorHandler extends LogLevelHandler {
    ErrorHandler() { super(LogLevel.ERROR); }
    void doHandle(LogRecord r, List<Appender> appenders) {
        appenders.forEach(a -> a.append(r));
        // The extensibility payoff: side effect lives here, not as a new branch in Logger.log().
        System.out.println("ALERT: on-call paged for ERROR: " + r.message);
    }
}
// INFO/WARN handlers omitted — identical shape to DebugHandler, no extra side effect (yet).

final class Logger {
    private final String name;
    private final LogLevel minLevel;
    private final List<Appender> appenders;
    private final Map<LogLevel, Integer> appenderMinLevelOverride; // optional per-appender floor, keyed illustratively by index in real impl

    Logger(String name, LogLevel minLevel, List<Appender> appenders) {
        this.name = name; this.minLevel = minLevel; this.appenders = appenders;
        this.appenderMinLevelOverride = Map.of();
    }

    void log(LogLevel level, String message) {
        if (level.ordinal() < minLevel.ordinal()) return; // logger-level filter
        appenders.forEach(a -> a.append(new LogRecord(level, message))); // per-appender filtering assumed pre-wired via decorators/config in a fuller impl
    }

    void debug(String msg) { log(LogLevel.DEBUG, msg); }
    void info(String msg)  { log(LogLevel.INFO, msg); }
    void warn(String msg)  { log(LogLevel.WARN, msg); }
    void error(String msg) { log(LogLevel.ERROR, msg); }
}

final class LoggerBuilder {
    private String name = "root";
    private LogLevel level = LogLevel.INFO;
    private final List<Appender> appenders = new ArrayList<>();

    LoggerBuilder named(String name) { this.name = name; return this; }
    LoggerBuilder withMinLevel(LogLevel level) { this.level = level; return this; }
    LoggerBuilder addAppender(Appender appender) { appenders.add(appender); return this; }

    Logger build() {
        if (appenders.isEmpty()) appenders.add(new ConsoleAppender()); // sane default
        return new Logger(name, level, appenders);
    }
}

// --- Singleton, justified: one process-wide named-logger registry ---
final class LoggerFactory {
    private static final LoggerFactory INSTANCE = new LoggerFactory();
    private final Map<String, Logger> registry = new ConcurrentHashMap<>();

    private LoggerFactory() {}
    static LoggerFactory getInstance() { return INSTANCE; }

    Logger getOrCreate(String name, java.util.function.Supplier<Logger> factory) {
        return registry.computeIfAbsent(name, n -> factory.get());
    }
}
```

## Python implementation

```python
from __future__ import annotations
from abc import ABC, abstractmethod
from concurrent.futures import ThreadPoolExecutor
from dataclasses import dataclass, field
from datetime import datetime
from enum import IntEnum, auto
from typing import Optional


class LogLevel(IntEnum):
    DEBUG = 10
    INFO = 20
    WARN = 30
    ERROR = 40


@dataclass
class LogRecord:
    level: LogLevel
    message: str
    timestamp: datetime = field(default_factory=datetime.utcnow)


class Appender(ABC):
    @abstractmethod
    def append(self, record: LogRecord) -> None: ...


class ConsoleAppender(Appender):
    def append(self, record: LogRecord) -> None:
        print(f"[{record.level.name}] {record.message}")


class FileAppender(Appender):
    def __init__(self, path: str):
        self.path = path

    def append(self, record: LogRecord) -> None:
        print(f"(file:{self.path}) [{record.level.name}] {record.message}")  # real impl: buffered write + rotation hook


class NetworkAppender(Appender):
    def __init__(self, endpoint: str):
        self.endpoint = endpoint

    def append(self, record: LogRecord) -> None:
        print(f"(POST {self.endpoint}) [{record.level.name}] {record.message}")


# --- Decorator: additive behavior, base appenders untouched ---
class AppenderDecorator(Appender):
    def __init__(self, delegate: Appender):
        self.delegate = delegate


class TimestampingAppender(AppenderDecorator):
    def append(self, record: LogRecord) -> None:
        stamped = LogRecord(record.level, f"[{record.timestamp.isoformat()}] {record.message}")
        self.delegate.append(stamped)


class AsyncAppender(AppenderDecorator):
    """Offloads the write to a single-worker pool — preserves order, non-blocking for the caller.
    Unbounded task submission can back up under sustained overload; see follow-ups."""
    def __init__(self, delegate: Appender):
        super().__init__(delegate)
        self._pool = ThreadPoolExecutor(max_workers=1)

    def append(self, record: LogRecord) -> None:
        self._pool.submit(self.delegate.append, record)


# --- Chain of Responsibility ---
class LogLevelHandler(ABC):
    def __init__(self, handles: LogLevel):
        self.handles = handles
        self.next: Optional["LogLevelHandler"] = None

    def set_next(self, nxt: "LogLevelHandler") -> "LogLevelHandler":
        self.next = nxt
        return nxt

    def handle(self, record: LogRecord, appenders: list[Appender]) -> None:
        if record.level == self.handles:
            self.do_handle(record, appenders)
        if self.next:
            self.next.handle(record, appenders)

    @abstractmethod
    def do_handle(self, record: LogRecord, appenders: list[Appender]) -> None: ...


class DebugHandler(LogLevelHandler):
    def __init__(self):
        super().__init__(LogLevel.DEBUG)

    def do_handle(self, record, appenders):
        for a in appenders:
            a.append(record)


class ErrorHandler(LogLevelHandler):
    def __init__(self):
        super().__init__(LogLevel.ERROR)

    def do_handle(self, record, appenders):
        for a in appenders:
            a.append(record)
        print(f"ALERT: on-call paged for ERROR: {record.message}")  # side effect lives in the handler, not in Logger.log


class Logger:
    def __init__(self, name: str, min_level: LogLevel, appenders: list[Appender]):
        self.name = name
        self.min_level = min_level
        self.appenders = appenders

    def log(self, level: LogLevel, message: str) -> None:
        if level < self.min_level:
            return
        record = LogRecord(level, message)
        for appender in self.appenders:
            appender.append(record)

    def debug(self, msg: str) -> None: self.log(LogLevel.DEBUG, msg)
    def info(self, msg: str) -> None: self.log(LogLevel.INFO, msg)
    def warn(self, msg: str) -> None: self.log(LogLevel.WARN, msg)
    def error(self, msg: str) -> None: self.log(LogLevel.ERROR, msg)


class LoggerBuilder:
    def __init__(self):
        self._name = "root"
        self._level = LogLevel.INFO
        self._appenders: list[Appender] = []

    def named(self, name: str) -> "LoggerBuilder":
        self._name = name
        return self

    def with_min_level(self, level: LogLevel) -> "LoggerBuilder":
        self._level = level
        return self

    def add_appender(self, appender: Appender) -> "LoggerBuilder":
        self._appenders.append(appender)
        return self

    def build(self) -> Logger:
        appenders = self._appenders or [ConsoleAppender()]
        return Logger(self._name, self._level, appenders)


class LoggerFactory:
    """Justified module-level Singleton: one process-wide named-logger registry.
    Python idiom: a module-level instance IS the Singleton — no need for the Java
    private-constructor dance; see ../04-python-oop-essentials.md for why."""
    _instance: Optional["LoggerFactory"] = None
    _registry: dict[str, Logger]

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._registry = {}
        return cls._instance

    def get_or_create(self, name: str, factory) -> Logger:
        if name not in self._registry:
            self._registry[name] = factory()
        return self._registry[name]
```

## Sample walkthrough

```python
factory = LoggerFactory()

app_logger = factory.get_or_create(
    "app",
    lambda: (
        LoggerBuilder()
        .named("app")
        .with_min_level(LogLevel.DEBUG)
        .add_appender(TimestampingAppender(ConsoleAppender()))
        .add_appender(AsyncAppender(FileAppender("/var/log/app.log")))
        .build()
    ),
)

app_logger.debug("cache miss for key=user:42")   # console only shows timestamped DEBUG; file gets it async
app_logger.error("payment gateway timeout")        # both appenders fire; a CoR-wired pipeline would also page on-call
```

## Follow-up questions

- **"Logging is blocking request latency under load — make it async end-to-end."** The `AsyncAppender` decorator already offloads a single appender; for the whole pipeline, put a bounded queue between `Logger.log()` and dispatch, with a worker pool draining it — reference [../06-concurrency-essentials.md](../06-concurrency-essentials.md) for `ExecutorService`/`BlockingQueue` sizing and backpressure policy (drop-oldest vs block-caller vs drop-newest).
- **"Add log rotation (size- or time-based) to `FileAppender`."** Purely internal to `FileAppender` — swap the raw file handle for a rotating-file-writer strategy; no other class in the design changes, which is the point of keeping rotation an `Appender` concern rather than a `Logger` concern.
- **"Different appenders need different minimum levels (e.g. console shows DEBUG+, network only ERROR+)."** Wrap each `Appender` in a small `LevelFilteringAppender` decorator holding its own threshold, composed the same way as `TimestampingAppender` — the `Decorator` choice pays for itself again here instead of adding a level field to every appender class.
- **"What if an appender throws (e.g. file handle closed, network down)?"** Wrap each `append()` call at the `Logger` dispatch site in a try/catch-per-appender so one failing sink doesn't take down the others or the caller's request thread; optionally add a `RetryingAppender` decorator for transient network failures (see the retry discussion in [10-notification-and-observer-system.md](10-notification-and-observer-system.md)).
- **"We now need structured (JSON) log output for a log-aggregation pipeline."** Introduce a `LogFormatter` interface (`format(LogRecord) -> String`), inject it into `Appender` implementations instead of hardcoding string concatenation — additive, doesn't touch `Logger` or the CoR chain.

## Common mistakes on this problem

- Implementing level filtering as a single `if (level >= threshold)` and stopping there when the interviewer explicitly asks for level-specific behavior (paging, metrics) — that's the signal to reach for Chain of Responsibility, not more `if` branches.
- Making `Logger` itself responsible for formatting, writing to disk, *and* deciding retry/async policy — a God Object that should instead delegate formatting to a `LogFormatter` and I/O + resilience to `Appender`/its decorators.
- Treating `LoggerFactory` as just "a Singleton because loggers are global," without articulating *why* multiple instances would be wrong (split registries → the same logger name resolves to different configs in different parts of the app) — say the justification out loud, don't just name the pattern.
- Building the async path with an unbounded queue/thread-per-log-call — this is the classic way a "make it async" follow-up turns into an OOM or thread-exhaustion incident; always reason about bounded queues and a fixed worker pool.

## Continue

Next: [10-notification-and-observer-system.md](10-notification-and-observer-system.md)
