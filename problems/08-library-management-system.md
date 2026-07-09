# Library Management System

## Requirements

- "How many copies can a title have?" — Multiple physical copies per title, each independently trackable (barcode, condition, current status). **You decide**: this is the central modeling lesson of this problem — split `Book` (catalog metadata) from `BookItem` (physical copy).
- "Do member types have different privileges?" — Yes: `Student` (5 books, 14-day loan), `Faculty` (10 books, 30-day loan), `Guest` (2 books, 7-day loan, no reservations). **You decide** if not specified: assume at least two tiers to justify polymorphism.
- "How are overdue fines computed?" — Assume a flat per-day rate that can vary by member tier or book category (reference books cost more per day late). Model as pluggable, not hardcoded.
- "What happens when a reserved book becomes available?" — Notify the first member in the reservation queue (FIFO); they get a hold window (e.g. 48 hours) before it goes to the next person. **You decide**: single library branch for the core design, multi-branch is a follow-up.
- "Is this single-threaded?" — Assume single-threaded core logic for the interview; note that checkout/reservation of the *same* `BookItem` needs a lock if concurrent (see [../06-concurrency-essentials.md](../06-concurrency-essentials.md)).

**In scope**: catalog (Book/BookItem), member management with tiers, checkout, return, reservation (hold queue), fine calculation, librarian actions (add/remove book, block member).

**Out of scope**: payments for fines (assume an external `PaymentGateway` stub), multi-branch inventory transfer, digital/e-book lending, search/indexing internals.

## Core entities & relationships

```
Library
  ├─ has-a[*] Book                     (catalog entry: title/author/ISBN — metadata only)
  │     └─ has-a[*] BookItem           (physical copy: barcode, condition, status)
  ├─ has-a[*] Member (interface: LibraryAccount)
  │     ├─ Student   (implements LibraryAccount)
  │     ├─ Faculty    (implements LibraryAccount)
  │     └─ Guest       (implements LibraryAccount)
  ├─ has-a[*] Librarian
  └─ has-a[1] FineCalculationStrategy (interface)

BookItem
  └─ has-a[1] BookItemStatus enum      (AVAILABLE | LOANED | RESERVED | LOST)

Loan
  ├─ has-a[1] BookItem
  ├─ has-a[1] Member
  └─ has-a[1] due date, [0..1] returned date

ReservationQueue (per Book, or per BookItem pool)
  └─ has-a[*] Member                   (FIFO hold queue)
```

**Why `Book` vs `BookItem` is a has-a[*], not a single class**: a `Book` is an idea ("Clean Code, 1st ed, ISBN X") — it never changes state. A `BookItem` is a physical, stateful object — it gets loaned, damaged, lost, and returned. Collapsing them forces you to duplicate title/author on every copy row and makes "how many copies of X are available" a filter query instead of a first-class relationship. This is the #1 thing interviewers watch for on this specific problem.

**Why `Member` is an interface/abstract type with tiers as subtypes, not a `type` enum field**: loan limit and loan duration differ by tier, but *no method's behavior changes in a surprising way* — `checkoutLimit()` and `loanDurationDays()` are just different return values, not different control flow. This is a clean [LSP](../02-solid-principles.md#l--liskov-substitution-principle) case: a `Faculty` is fully substitutable anywhere a `Member`/`LibraryAccount` is expected — nothing overridden throws or contradicts the base contract, so subtyping is safe here (contrast with the `ReadOnlyAccount extends Account` anti-example in the SOLID doc). If a tier needed genuinely different *behavior* (e.g. Guests can't reserve at all), that's still fine as an overridden method returning `false`/no-op, not a thrown exception — the contract stays honest.

**Why reservation is a queue keyed per title, not per copy**: a member reserving "Clean Code" doesn't care which physical copy they get, only that *a* copy becomes available — so the hold queue lives on `Book` and hands off to whichever `BookItem` becomes `AVAILABLE` first.

## Design patterns applied

- [Strategy](../patterns/03-behavioral-patterns.md#strategy) — `FineCalculationStrategy` varies fine-per-day by member tier or book category; new fine schemes (e.g. capped fine, grace period) are new classes, not edits to a `calculateFine` if-chain.
- [Observer](../patterns/03-behavioral-patterns.md#observer) — when a `BookItem` transitions to `AVAILABLE`, the `Book`'s reservation queue is notified so the head-of-queue member gets a hold notice; decouples inventory state changes from the notification mechanism.
- [Factory Method](../patterns/01-creational-patterns.md#factory-method) — `MemberFactory.create(MemberType, ...)` centralizes which concrete `Member` subtype (and its loan-limit policy) to instantiate, so account-creation logic isn't duplicated at every call site that signs up a member.
- [Template Method](../patterns/03-behavioral-patterns.md#template-method) *(optional, if probed)* — a shared `checkout()` skeleton (validate limit → create loan → mark item loaned → log) with tier-specific hook steps, if tiers ever need extra checkout-time validation.

## Java implementation

```java
import java.time.LocalDate;
import java.util.*;

enum BookItemStatus { AVAILABLE, LOANED, RESERVED, LOST }

final class Book {
    private final String isbn;
    private final String title;
    private final String author;
    private final List<BookItem> copies = new ArrayList<>();
    private final Deque<Member> reservationQueue = new LinkedList<>(); // FIFO hold queue

    Book(String isbn, String title, String author) {
        this.isbn = isbn; this.title = title; this.author = author;
    }

    void addCopy(BookItem item) { copies.add(item); }

    Optional<BookItem> firstAvailableCopy() {
        return copies.stream().filter(c -> c.getStatus() == BookItemStatus.AVAILABLE).findFirst();
    }

    void enqueueReservation(Member m) { reservationQueue.addLast(m); }

    // Called by the observer callback when a copy frees up.
    Optional<Member> nextInQueue() {
        return Optional.ofNullable(reservationQueue.pollFirst());
    }

    String getIsbn() { return isbn; }
    String getTitle() { return title; }
}

final class BookItem {
    private final String barcode;
    private final Book book;
    private BookItemStatus status = BookItemStatus.AVAILABLE;
    private final List<AvailabilityListener> listeners = new ArrayList<>(); // Observer

    BookItem(String barcode, Book book) { this.barcode = barcode; this.book = book; }

    void addAvailabilityListener(AvailabilityListener l) { listeners.add(l); }

    void markLoaned() { status = BookItemStatus.LOANED; }

    void markReturned() {
        status = BookItemStatus.AVAILABLE;
        // Notify observers *before* anyone else can grab this copy — avoids a lost-wakeup race.
        for (AvailabilityListener l : listeners) l.onAvailable(book, this);
    }

    void markLost() { status = BookItemStatus.LOST; }

    BookItemStatus getStatus() { return status; }
    String getBarcode() { return barcode; }
    Book getBook() { return book; }
}

// Observer interface — a copy becoming available is the "subject state change" this problem's
// canonical Observer instance models; see 10-notification-and-observer-system.md for the deep dive.
interface AvailabilityListener {
    void onAvailable(Book book, BookItem item);
}

interface FineCalculationStrategy {
    double calculateFine(Loan loan, LocalDate returnDate);
}

final class FlatDailyFineStrategy implements FineCalculationStrategy {
    private final double perDayRate;
    FlatDailyFineStrategy(double perDayRate) { this.perDayRate = perDayRate; }

    public double calculateFine(Loan loan, LocalDate returnDate) {
        long overdueDays = Math.max(0, returnDate.toEpochDay() - loan.getDueDate().toEpochDay());
        return overdueDays * perDayRate;
    }
}

interface LibraryAccount {
    String getMemberId();
    int checkoutLimit();
    int loanDurationDays();
    boolean canReserve();
}

abstract class Member implements LibraryAccount {
    private final String memberId;
    private final List<Loan> activeLoans = new ArrayList<>();

    Member(String memberId) { this.memberId = memberId; }

    public String getMemberId() { return memberId; }
    List<Loan> getActiveLoans() { return activeLoans; }
    boolean canCheckoutMore() { return activeLoans.size() < checkoutLimit(); }
}

final class Student extends Member {
    Student(String id) { super(id); }
    public int checkoutLimit() { return 5; }
    public int loanDurationDays() { return 14; }
    public boolean canReserve() { return true; }
}

final class Faculty extends Member {
    Faculty(String id) { super(id); }
    public int checkoutLimit() { return 10; }
    public int loanDurationDays() { return 30; }
    public boolean canReserve() { return true; }
}

final class Guest extends Member {
    Guest(String id) { super(id); }
    public int checkoutLimit() { return 2; }
    public int loanDurationDays() { return 7; }
    public boolean canReserve() { return false; } // guests opt out of the reservation queue entirely
}

enum MemberType { STUDENT, FACULTY, GUEST }

final class MemberFactory {
    static Member create(MemberType type, String id) {
        return switch (type) {
            case STUDENT -> new Student(id);
            case FACULTY -> new Faculty(id);
            case GUEST -> new Guest(id);
        };
    }
}

final class Loan {
    private final BookItem item;
    private final Member member;
    private final LocalDate dueDate;
    private LocalDate returnedDate; // null while active

    Loan(BookItem item, Member member, LocalDate dueDate) {
        this.item = item; this.member = member; this.dueDate = dueDate;
    }

    LocalDate getDueDate() { return dueDate; }
    void markReturned(LocalDate date) { this.returnedDate = date; }
    boolean isActive() { return returnedDate == null; }
    BookItem getItem() { return item; }
    Member getMember() { return member; }
}

final class Library {
    private final Map<String, Book> catalogByIsbn = new HashMap<>();
    private final FineCalculationStrategy fineStrategy;

    Library(FineCalculationStrategy fineStrategy) { this.fineStrategy = fineStrategy; }

    void addBook(Book book) { catalogByIsbn.put(book.getIsbn(), book); }

    Loan checkout(Member member, String isbn, LocalDate today) {
        if (!member.canCheckoutMore())
            throw new IllegalStateException("Checkout limit reached for " + member.getMemberId());

        Book book = catalogByIsbn.get(isbn);
        BookItem item = book.firstAvailableCopy()
            .orElseThrow(() -> new NoSuchElementException("No available copy of " + isbn));

        item.markLoaned();
        Loan loan = new Loan(item, member, today.plusDays(member.loanDurationDays()));
        member.getActiveLoans().add(loan);

        // Wire the hold-queue hand-off: when THIS item frees up again, offer it to next in queue.
        item.addAvailabilityListener((b, freedItem) ->
            b.nextInQueue().ifPresent(next -> System.out.println(
                "Notify " + next.getMemberId() + ": '" + b.getTitle() + "' is available for pickup")));
        return loan;
    }

    double returnBook(Loan loan, LocalDate today) {
        loan.markReturned(today);
        loan.getMember().getActiveLoans().remove(loan);
        loan.getItem().markReturned(); // fires AvailabilityListener -> reservation hand-off
        return fineStrategy.calculateFine(loan, today);
    }

    void reserve(Member member, String isbn) {
        if (!member.canReserve())
            throw new UnsupportedOperationException(member.getMemberId() + " tier cannot reserve");
        catalogByIsbn.get(isbn).enqueueReservation(member);
    }
}
```

## Python implementation

```python
from __future__ import annotations
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from datetime import date, timedelta
from enum import Enum, auto
from collections import deque
from typing import Callable, Optional


class BookItemStatus(Enum):
    AVAILABLE = auto()
    LOANED = auto()
    RESERVED = auto()
    LOST = auto()


@dataclass
class Book:
    isbn: str
    title: str
    author: str
    copies: list["BookItem"] = field(default_factory=list)
    _reservation_queue: deque["Member"] = field(default_factory=deque)

    def add_copy(self, item: "BookItem") -> None:
        self.copies.append(item)

    def first_available_copy(self) -> Optional["BookItem"]:
        return next((c for c in self.copies if c.status == BookItemStatus.AVAILABLE), None)

    def enqueue_reservation(self, member: "Member") -> None:
        self._reservation_queue.append(member)

    def next_in_queue(self) -> Optional["Member"]:
        return self._reservation_queue.popleft() if self._reservation_queue else None


AvailabilityListener = Callable[[Book, "BookItem"], None]


class BookItem:
    def __init__(self, barcode: str, book: Book):
        self.barcode = barcode
        self.book = book
        self.status = BookItemStatus.AVAILABLE
        self._listeners: list[AvailabilityListener] = []

    def add_availability_listener(self, listener: AvailabilityListener) -> None:
        self._listeners.append(listener)

    def mark_loaned(self) -> None:
        self.status = BookItemStatus.LOANED

    def mark_returned(self) -> None:
        self.status = BookItemStatus.AVAILABLE
        for listener in self._listeners:
            listener(self.book, self)

    def mark_lost(self) -> None:
        self.status = BookItemStatus.LOST


class FineCalculationStrategy(ABC):
    @abstractmethod
    def calculate_fine(self, loan: "Loan", return_date: date) -> float: ...


class FlatDailyFineStrategy(FineCalculationStrategy):
    def __init__(self, per_day_rate: float):
        self.per_day_rate = per_day_rate

    def calculate_fine(self, loan: "Loan", return_date: date) -> float:
        overdue_days = max(0, (return_date - loan.due_date).days)
        return overdue_days * self.per_day_rate


class Member(ABC):
    def __init__(self, member_id: str):
        self.member_id = member_id
        self.active_loans: list["Loan"] = []

    @property
    @abstractmethod
    def checkout_limit(self) -> int: ...

    @property
    @abstractmethod
    def loan_duration_days(self) -> int: ...

    @property
    @abstractmethod
    def can_reserve(self) -> bool: ...

    def can_checkout_more(self) -> bool:
        return len(self.active_loans) < self.checkout_limit


class Student(Member):
    checkout_limit = 5
    loan_duration_days = 14
    can_reserve = True


class Faculty(Member):
    checkout_limit = 10
    loan_duration_days = 30
    can_reserve = True


class Guest(Member):
    checkout_limit = 2
    loan_duration_days = 7
    can_reserve = False  # opts out of the hold queue entirely, mirrors the Java UnsupportedOperationException path


class MemberType(Enum):
    STUDENT = "student"
    FACULTY = "faculty"
    GUEST = "guest"


def create_member(member_type: MemberType, member_id: str) -> Member:
    return {
        MemberType.STUDENT: Student,
        MemberType.FACULTY: Faculty,
        MemberType.GUEST: Guest,
    }[member_type](member_id)


@dataclass
class Loan:
    item: BookItem
    member: Member
    due_date: date
    returned_date: Optional[date] = None

    @property
    def is_active(self) -> bool:
        return self.returned_date is None


class Library:
    def __init__(self, fine_strategy: FineCalculationStrategy):
        self.fine_strategy = fine_strategy
        self._catalog: dict[str, Book] = {}

    def add_book(self, book: Book) -> None:
        self._catalog[book.isbn] = book

    def checkout(self, member: Member, isbn: str, today: date) -> Loan:
        if not member.can_checkout_more():
            raise RuntimeError(f"Checkout limit reached for {member.member_id}")

        book = self._catalog[isbn]
        item = book.first_available_copy()
        if item is None:
            raise LookupError(f"No available copy of {isbn}")

        item.mark_loaned()
        loan = Loan(item, member, today + timedelta(days=member.loan_duration_days))
        member.active_loans.append(loan)

        def on_available(b: Book, freed_item: BookItem) -> None:
            nxt = b.next_in_queue()
            if nxt:
                print(f"Notify {nxt.member_id}: '{b.title}' is available for pickup")

        item.add_availability_listener(on_available)
        return loan

    def return_book(self, loan: Loan, today: date) -> float:
        loan.returned_date = today
        loan.member.active_loans.remove(loan)
        loan.item.mark_returned()  # fires listener -> reservation hand-off
        return self.fine_strategy.calculate_fine(loan, today)

    def reserve(self, member: Member, isbn: str) -> None:
        if not member.can_reserve:
            raise PermissionError(f"{member.member_id} tier cannot reserve")
        self._catalog[isbn].enqueue_reservation(member)
```

## Sample walkthrough

```python
library = Library(FlatDailyFineStrategy(per_day_rate=0.50))

book = Book(isbn="978-0132350884", title="Clean Code", author="Robert C. Martin")
book.add_copy(BookItem("CC-001", book))
library.add_book(book)

alice = create_member(MemberType.STUDENT, "alice")
bob = create_member(MemberType.STUDENT, "bob")

loan = library.checkout(alice, "978-0132350884", today=date(2026, 6, 1))
library.reserve(bob, "978-0132350884")           # only copy is out, bob queues

fine = library.return_book(loan, today=date(2026, 6, 20))
# -> prints "Notify bob: 'Clean Code' is available for pickup"
print(f"Fine owed: ${fine:.2f}")                 # 5 days overdue * $0.50 = $2.50
```

## Follow-up questions

- **"Now support multiple branches, each with its own copies."** Add a `Branch` entity that owns a subset of `BookItem`s for a `Book`; `firstAvailableCopy()` becomes branch-scoped (or branch-preferred with fallback). No change to `Member`, `Loan`, or the fine strategy — the change is additive at the `Book`↔`BookItem` join.
- **"What if two members try to check out the last copy at the same instant?"** Single-threaded logic above has a race on `firstAvailableCopy()` + `markLoaned()`. Fix: guard per-`BookItem` (or per-`Book`) with a lock, or make the check-and-mark one atomic operation (`compareAndSet`-style status transition). See [../06-concurrency-essentials.md](../06-concurrency-essentials.md) for `ReentrantLock` vs `synchronized` trade-offs.
- **"Add a hold expiration — if the notified member doesn't pick up within 48 hours, offer it to the next person."** Store a hold-expiry timestamp on the head of the queue when notified; a scheduled sweep (or lazy check on next queue access) pops expired holds and re-notifies. Purely additive to `Book`'s queue handling.
- **"Support e-books with no physical inventory / infinite copies."** Introduce a `LendingPolicy` interface: `PhysicalLendingPolicy` (finite copies, uses `BookItem`) vs `DigitalLendingPolicy` (concurrent-license-capped or infinite). `Book` delegates availability checks to its policy instead of assuming `BookItem` always applies.
- **"Different fine caps per member tier (e.g. Faculty fines cap at $20)."** Wrap the existing `FineCalculationStrategy` with a `CappedFineStrategy` decorator, or make cap a parameter on the strategy — no change to `Library.returnBook`.

## Common mistakes on this problem

- Collapsing `Book` and `BookItem` into one class with a `copiesAvailable: int` counter — this loses per-copy state (which specific copy is damaged/lost/loaned to whom) and is the single most common critique on this problem.
- Modeling member tiers as an `int discountLevel` field or `type` enum checked via `if/else` in `checkout()`, instead of letting `checkoutLimit()`/`loanDurationDays()` be polymorphic — reintroduces the OCP violation the tier hierarchy exists to avoid.
- Putting the reservation queue on `BookItem` (the copy) instead of `Book` (the title) — a member reserving a title shouldn't care which physical barcode eventually satisfies it; queueing per-copy causes members to wait behind a specific copy while other copies of the same title sit available.
- Forgetting fines are computed at *return* time against the loan's due date, and instead trying to track "days late" incrementally while the loan is active — needless state; it's a pure function of `(dueDate, returnDate)`.

## Continue

Next: [09-logging-framework.md](09-logging-framework.md)
