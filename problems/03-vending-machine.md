# Vending Machine

The cleanest possible showcase for [State](../patterns/03-behavioral-patterns.md#state): four states, each with obviously different behavior for the same two triggering actions (select item, insert money), and a transition graph you can draw in 30 seconds. If you can't explain State cleanly on this problem, revisit it before the interview — every other State-pattern problem is a variation on this shape.

## Requirements

- "What can go wrong that the machine must handle gracefully?" → Invalid selection, insufficient funds, item out of stock, and the machine being unable to make exact change — **you decide** to model all four explicitly rather than hand-wave "assume happy path," since this is precisely what the State machine is for.
- "Cash only, or cards too?" → Cash (coins + notes) for the core design; **you decide** to model payment as accepted denominations for now and flag card payment as a `PaymentMethod` Strategy extension in follow-ups, not core scope.
- "Multiple items per slot, or one-of-each?" → Each slot holds a quantity of one `Item`; multiple slots can hold the same item code (real machines do this) — out of scope to optimize slot selection, just decrement the first slot with stock.
- "Does the machine ever fully run out and stop accepting money?" → Yes — machine-wide `OutOfStock` (all slots empty) is a state, not just a per-item error, per the problem's own naming.
- "Refunds — full transaction cancel only, or partial?" → **You decide**: full refund of inserted money only, no partial dispense-and-partial-refund logic.
- "Restocking — who does it and when?" → An admin-only `restock()` operation that isn't part of the customer-facing state transitions (modeled as a direct call on `VendingMachine`, not a `VendingMachineState` method) — out of scope to model an admin role/auth.

**In scope:** item selection, money insertion (cumulative, can under-pay across multiple inserts), exact-change dispensing via a pluggable algorithm, out-of-stock detection at both item and machine level, refund.

**Out of scope:** card/UPI payment integration, multi-currency, partial dispense, admin authentication, inventory replenishment scheduling.

## Core entities & relationships

```
VendingMachine
  ├─ has-a[1] VendingMachineState (interface)
  ├─ has-a[*] Slot
  └─ has-a[1] ChangeCalculator (interface)

Slot
  └─ has-a[1] Item + quantity
```

`VendingMachineState` is the textbook GoF State: `Idle`, `HasMoney`, `Dispensing`, `OutOfStock` each react *differently* to the same two public triggers (`selectItem`, `insertMoney`) — `Idle` requires a selection before accepting money, `HasMoney` accumulates and auto-transitions once sufficient, `Dispensing` is a transient state that rejects new input mid-transaction, `OutOfStock` rejects everything until an out-of-band `restock()`. Contrast with the parking lot's `VehicleType`, which is an enum precisely *because* it has no transition logic — here the transition logic is the entire point.

`Slot` is composition (a `VendingMachine` without slots is meaningless and slots don't outlive the machine); `Item` is a plain immutable value object (code/name/price) referenced by possibly-multiple slots, so it's aggregation from the slot's side.

## Design patterns applied

- [State](../patterns/03-behavioral-patterns.md#state) — the primary lesson here: `Idle → HasMoney → Dispensing → (Idle | OutOfStock)` as explicit classes means "dispense while no money inserted" or "accept money while dispensing" are structurally unreachable, not bugs you're hoping your `if` chain prevents.
- [Strategy](../patterns/03-behavioral-patterns.md#strategy) — `ChangeCalculator` isolates the change-making algorithm (greedy largest-denomination-first today) from the state machine; swapping in an algorithm that accounts for scarce denominations (see follow-ups) touches one class.

## Java implementation

```java
enum Denomination {
    PENNY(1), NICKEL(5), DIME(10), QUARTER(25), ONE_DOLLAR(100), FIVE_DOLLAR(500);
    final int cents;
    Denomination(int cents) { this.cents = cents; }
}

record Item(String code, String name, int priceCents) {}

final class Slot {
    private final Item item;
    private int quantity;
    Slot(Item item, int quantity) { this.item = item; this.quantity = quantity; }
    boolean inStock() { return quantity > 0; }
    void decrement() { quantity--; }
    void restock(int count) { quantity += count; }
    Item getItem() { return item; }
}

interface ChangeCalculator {
    Map<Denomination, Integer> makeChange(int amountCents, Map<Denomination, Integer> available);
}

final class GreedyChangeCalculator implements ChangeCalculator {
    public Map<Denomination, Integer> makeChange(int amountCents, Map<Denomination, Integer> available) {
        Map<Denomination, Integer> result = new LinkedHashMap<>();
        Denomination[] descending = Arrays.stream(Denomination.values())
            .sorted(Comparator.comparingInt((Denomination d) -> d.cents).reversed())
            .toArray(Denomination[]::new);
        int remaining = amountCents;
        for (Denomination d : descending) {
            int have = available.getOrDefault(d, 0);
            int use = Math.min(have, remaining / d.cents);
            if (use > 0) { result.put(d, use); remaining -= use * d.cents; }
        }
        if (remaining != 0) throw new IllegalStateException("Cannot make exact change"); // see Follow-up
        return result;
    }
}

interface VendingMachineState {
    void selectItem(VendingMachine m, String code);
    void insertMoney(VendingMachine m, Denomination d);
    void refund(VendingMachine m);
}

final class IdleState implements VendingMachineState {
    public void selectItem(VendingMachine m, String code) {
        Slot slot = m.findSlot(code);
        if (slot == null || !slot.inStock()) throw new IllegalArgumentException("Unavailable: " + code);
        m.setSelectedSlot(slot);
    }
    public void insertMoney(VendingMachine m, Denomination d) {
        if (m.getSelectedSlot() == null) throw new IllegalStateException("Select an item first");
        m.addBalance(d);
        m.setState(new HasMoneyState());
        m.getState().insertMoney(m, null); // re-dispatch: let HasMoney decide if that was already enough (see note below)
    }
    public void refund(VendingMachine m) { /* nothing inserted yet */ }
}

final class HasMoneyState implements VendingMachineState {
    public void selectItem(VendingMachine m, String code) {
        throw new IllegalStateException("Finish or cancel current transaction first");
    }
    public void insertMoney(VendingMachine m, Denomination d) {
        if (d != null) m.addBalance(d); // null means "re-check after Idle's initial insert", see caller
        if (m.getBalanceCents() >= m.getSelectedSlot().getItem().priceCents()) {
            m.setState(new DispensingState());
            m.getState().onEnter(m); // Dispensing has no public trigger; it runs to completion synchronously
        }
    }
    public void refund(VendingMachine m) {
        m.refundBalance();
        m.reset();
        m.setState(new IdleState());
    }
}

final class DispensingState implements VendingMachineState {
    void onEnter(VendingMachine m) {
        Slot slot = m.getSelectedSlot();
        slot.decrement();
        int change = m.getBalanceCents() - slot.getItem().priceCents();
        Map<Denomination, Integer> changeGiven = m.getChangeCalculator().makeChange(change, m.getCashInventory());
        m.dispenseItem(slot.getItem(), changeGiven);
        m.reset();
        m.setState(m.hasAnyStock() ? new IdleState() : new OutOfStockState());
    }
    public void selectItem(VendingMachine m, String code) { throw new IllegalStateException("Dispensing in progress"); }
    public void insertMoney(VendingMachine m, Denomination d) { throw new IllegalStateException("Dispensing in progress"); }
    public void refund(VendingMachine m) { throw new IllegalStateException("Dispensing in progress"); }
}

final class OutOfStockState implements VendingMachineState {
    public void selectItem(VendingMachine m, String code) { throw new IllegalStateException("Machine out of stock"); }
    public void insertMoney(VendingMachine m, Denomination d) { throw new IllegalStateException("Machine out of stock"); }
    public void refund(VendingMachine m) { /* no balance can exist in this state */ }
}

final class VendingMachine {
    private final Map<String, Slot> slots = new HashMap<>();
    private final Map<Denomination, Integer> cashInventory = new EnumMap<>(Denomination.class);
    private final ChangeCalculator changeCalculator;
    private VendingMachineState state = new IdleState();
    private Slot selectedSlot;
    private int balanceCents;

    VendingMachine(List<Slot> initialSlots, ChangeCalculator changeCalculator) {
        for (Slot s : initialSlots) slots.put(s.getItem().code(), s);
        this.changeCalculator = changeCalculator;
    }

    void selectItem(String code) { state.selectItem(this, code); }
    void insertMoney(Denomination d) { state.insertMoney(this, d); }
    void refund() { state.refund(this); }
    void restock(String code, int count) {
        slots.get(code).restock(count);
        if (state instanceof OutOfStockState) state = new IdleState();
    }

    // package-private helpers used only by state classes
    Slot findSlot(String code) { return slots.get(code); }
    void setSelectedSlot(Slot s) { selectedSlot = s; }
    Slot getSelectedSlot() { return selectedSlot; }
    void addBalance(Denomination d) { balanceCents += d.cents; cashInventory.merge(d, 1, Integer::sum); }
    int getBalanceCents() { return balanceCents; }
    void refundBalance() { /* return balanceCents to customer via cashInventory reversal — omitted, symmetric to dispenseItem */ }
    void reset() { selectedSlot = null; balanceCents = 0; }
    boolean hasAnyStock() { return slots.values().stream().anyMatch(Slot::inStock); }
    void setState(VendingMachineState s) { this.state = s; }
    VendingMachineState getState() { return state; }
    ChangeCalculator getChangeCalculator() { return changeCalculator; }
    Map<Denomination, Integer> getCashInventory() { return cashInventory; }
    void dispenseItem(Item item, Map<Denomination, Integer> change) {
        System.out.println("Dispensing " + item.name() + ", change: " + change);
    }
}
```

**Note on the `Idle → HasMoney` re-dispatch:** `IdleState.insertMoney` immediately transitions to `HasMoneyState` and re-invokes `insertMoney` so a single sufficient payment doesn't require a second call — a slightly awkward re-dispatch that's worth calling out as a design wart in the interview and offering the cleaner fix: give every state an `onEnter(VendingMachine)` hook (like `DispensingState` already has) that re-checks sufficiency immediately on transition, rather than special-casing inside `insertMoney`.

## Python implementation

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from enum import Enum

class Denomination(Enum):
    PENNY = 1; NICKEL = 5; DIME = 10; QUARTER = 25; ONE_DOLLAR = 100; FIVE_DOLLAR = 500

@dataclass(frozen=True)
class Item:
    code: str
    name: str
    price_cents: int

class Slot:
    def __init__(self, item: Item, quantity: int):
        self.item = item
        self.quantity = quantity
    def in_stock(self) -> bool:
        return self.quantity > 0
    def decrement(self) -> None:
        self.quantity -= 1
    def restock(self, count: int) -> None:
        self.quantity += count

class ChangeCalculator(ABC):
    @abstractmethod
    def make_change(self, amount_cents: int, available: dict[Denomination, int]) -> dict[Denomination, int]: ...

class GreedyChangeCalculator(ChangeCalculator):
    def make_change(self, amount_cents, available):
        result: dict[Denomination, int] = {}
        remaining = amount_cents
        for d in sorted(Denomination, key=lambda x: -x.value):
            have = available.get(d, 0)
            use = min(have, remaining // d.value)
            if use:
                result[d] = use
                remaining -= use * d.value
        if remaining != 0:
            raise RuntimeError("Cannot make exact change")
        return result

class VendingMachineState(ABC):
    @abstractmethod
    def select_item(self, m: "VendingMachine", code: str) -> None: ...
    @abstractmethod
    def insert_money(self, m: "VendingMachine", d: Denomination | None) -> None: ...
    @abstractmethod
    def refund(self, m: "VendingMachine") -> None: ...

class IdleState(VendingMachineState):
    def select_item(self, m, code):
        slot = m.find_slot(code)
        if not slot or not slot.in_stock():
            raise ValueError(f"Unavailable: {code}")
        m.selected_slot = slot
    def insert_money(self, m, d):
        if m.selected_slot is None:
            raise RuntimeError("Select an item first")
        m.add_balance(d)
        m.state = HasMoneyState()
        m.state.insert_money(m, None)  # re-check sufficiency immediately, see Java note
    def refund(self, m):
        pass

class HasMoneyState(VendingMachineState):
    def select_item(self, m, code):
        raise RuntimeError("Finish or cancel current transaction first")
    def insert_money(self, m, d):
        if d is not None:
            m.add_balance(d)
        if m.balance_cents >= m.selected_slot.item.price_cents:
            m.state = DispensingState()
            m.state.on_enter(m)
    def refund(self, m):
        m.refund_balance()
        m.reset()
        m.state = IdleState()

class DispensingState(VendingMachineState):
    def on_enter(self, m: "VendingMachine") -> None:
        slot = m.selected_slot
        slot.decrement()
        change_due = m.balance_cents - slot.item.price_cents
        change_given = m.change_calculator.make_change(change_due, m.cash_inventory)
        m.dispense_item(slot.item, change_given)
        m.reset()
        m.state = IdleState() if m.has_any_stock() else OutOfStockState()
    def select_item(self, m, code): raise RuntimeError("Dispensing in progress")
    def insert_money(self, m, d): raise RuntimeError("Dispensing in progress")
    def refund(self, m): raise RuntimeError("Dispensing in progress")

class OutOfStockState(VendingMachineState):
    def select_item(self, m, code): raise RuntimeError("Machine out of stock")
    def insert_money(self, m, d): raise RuntimeError("Machine out of stock")
    def refund(self, m): pass  # no balance can exist in this state

class VendingMachine:
    def __init__(self, initial_slots: list[Slot], change_calculator: ChangeCalculator):
        self.slots: dict[str, Slot] = {s.item.code: s for s in initial_slots}
        self.cash_inventory: dict[Denomination, int] = {}
        self.change_calculator = change_calculator
        self.state: VendingMachineState = IdleState()
        self.selected_slot: Slot | None = None
        self.balance_cents = 0

    def select_item(self, code: str) -> None: self.state.select_item(self, code)
    def insert_money(self, d: Denomination) -> None: self.state.insert_money(self, d)
    def refund(self) -> None: self.state.refund(self)
    def restock(self, code: str, count: int) -> None:
        self.slots[code].restock(count)
        if isinstance(self.state, OutOfStockState):
            self.state = IdleState()

    def find_slot(self, code: str) -> Slot | None: return self.slots.get(code)
    def add_balance(self, d: Denomination) -> None:
        self.balance_cents += d.value
        self.cash_inventory[d] = self.cash_inventory.get(d, 0) + 1
    def refund_balance(self) -> None: pass  # symmetric to dispense_item, omitted
    def reset(self) -> None: self.selected_slot = None; self.balance_cents = 0
    def has_any_stock(self) -> bool: return any(s.in_stock() for s in self.slots.values())
    def dispense_item(self, item: Item, change: dict[Denomination, int]) -> None:
        print(f"Dispensing {item.name}, change: {change}")
```

## Sample walkthrough

```python
cola = Item("A1", "Cola", 125)
machine = VendingMachine([Slot(cola, quantity=2)], GreedyChangeCalculator())

machine.select_item("A1")            # Idle.select_item -> selected_slot = cola slot
machine.insert_money(Denomination.ONE_DOLLAR)   # Idle -> HasMoney, balance = 100, not enough (125 needed)
machine.insert_money(Denomination.QUARTER)      # HasMoney: balance = 125 == price -> Dispensing.on_enter fires
                                                 # -> dispenses Cola, change = {} (exact), state -> Idle (1 left in stock)
```

## Follow-up questions

- **"What if the machine can't make exact change?"** `GreedyChangeCalculator.makeChange` already throws when `remaining != 0`; wire that exception in `DispensingState.onEnter` to trigger a refund-and-abort path instead of a dispense (transition to a `HasMoneyState` with the money still held, or straight to `IdleState` after auto-refund) — the fix is a `try/catch` at the one call site, the state machine shape doesn't change.
- **"Add card payment alongside cash."** Introduce a `PaymentMethod` interface (`cash` today, `card` tomorrow) that `HasMoneyState` delegates to for "is this transaction fully paid" instead of comparing `balanceCents` directly — this is the same DIP move as the parking lot's `PaymentMethod`, and doesn't touch the State hierarchy at all since payment method is orthogonal to transaction state.
- **"Two customers use the machine at the same "moment" (multi-machine fleet with shared inventory backend)?"** A single physical machine is inherently single-transaction — the real question is usually about a shared backend tracking inventory across many machines, which is a different service boundary; within one `VendingMachine`, guard `state`/`selectedSlot`/`balanceCents` mutations with a lock per [../06-concurrency-essentials.md](../06-concurrency-essentials.md) if the same machine is somehow driven by two threads (e.g., a touchscreen thread and a coin-sensor interrupt thread).
- **"Support combo deals (buy 2 get 1 free) or discounts?"** Add a `PricingPolicy`/discount Strategy consulted in `HasMoneyState.insertMoney` before comparing balance to price — again additive, doesn't touch `IdleState`/`DispensingState`/`OutOfStockState`.
- **"What if a single slot runs out mid-selection but others still have stock?"** Already handled: `OutOfStockState` is machine-wide (`hasAnyStock()` across *all* slots), while a single empty slot is just rejected at `selectItem` time (`slot.inStock()` check in `IdleState`) — the two failure modes are deliberately different states/checks, worth narrating why.

## Common mistakes on this problem

- Modeling machine status as a `String status` field with values like `"idle"`/`"has_money"` checked via `if (status.equals("idle"))` chains — loses compile-time safety and is the exact anti-pattern State replaces.
- Letting `DispensingState` be a public, externally-triggerable state (e.g., a public `dispense()` method callers invoke directly) instead of an internal transition — callers should never need to know dispensing is a distinct state; it's an implementation detail of the transaction.
- Conflating "this item's slot is empty" with "the machine is out of stock" — collapsing both into one `OutOfStock` state produces wrong behavior (rejecting selection of an item that's still available in another slot).
- Doing change calculation inline inside the state class instead of extracting `ChangeCalculator` — works fine until the interviewer asks "what if we run out of quarters," and now you're refactoring under time pressure instead of swapping a strategy.

## Continue

Next: [04-tic-tac-toe-and-chess.md](04-tic-tac-toe-and-chess.md)
