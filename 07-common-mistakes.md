# Common Mistakes That Lose LLD Interview Points

Read this right before any interview. Every item here maps to a scoring axis in [00-evaluation-framework.md](00-evaluation-framework.md) — these are the specific, recurring ways candidates lose points on each axis, not generic advice.

## Process mistakes (lose points before you write a line of code)

- **Diving into code with zero clarifying questions.** Even a strong design gets marked down if the interviewer can't see you scope the problem. Always run the script in [01-requirements-and-uml.md](01-requirements-and-uml.md), even briefly.
- **Over-spending the talk phase.** 25 minutes of requirements + diagram, 10 minutes of rushed code is worse than a slightly-less-polished diagram and complete, clean code. Respect the time budget table in [00-evaluation-framework.md](00-evaluation-framework.md).
- **Silent design.** Working through the problem without narrating WHY. The interviewer can only score what they can observe — say the SOLID/pattern reasoning out loud as you go (see the narration habit).
- **Not stating assumptions.** "I'll assume single-threaded for now and mention how I'd extend it" is a strong sentence. Silently assuming and never mentioning it looks like you didn't consider it at all.

## Modeling mistakes

- **God Object.** One class (`ParkingLot`, `Game`, `System`) that owns pricing, persistence, notification, and validation. Fix: nouns-and-verbs pass, split by axis of change ([SRP](02-solid-principles.md#s--single-responsibility-principle)).
- **`if/else` or `switch` on a type field, in business logic, that will need a new branch for every new variant.** This is the single most commonly flagged issue. Fix: polymorphism / [Strategy](patterns/03-behavioral-patterns.md#strategy) ([OCP](02-solid-principles.md#o--openclosed-principle)).
- **Modeling booleans instead of an explicit state enum/machine** (`isMoving`, `doorsOpen`, `isDispensing` as separate booleans that can go out of sync) when the problem is inherently state-machine-shaped (elevator, vending machine, seat, order). Fix: [State pattern](patterns/03-behavioral-patterns.md#state).
- **Forcing an is-a relationship that doesn't hold** just to reuse code, then overriding methods to throw `UnsupportedOperationException`/`NotImplementedError`. This is an [LSP](02-solid-principles.md#l--liskov-substitution-principle) violation — restructure the hierarchy (extract a capability interface) instead.
- **Fat interfaces** that force unrelated implementers to stub out methods they don't need. Fix: [ISP](02-solid-principles.md#i--interface-segregation-principle) — split into role interfaces.
- **Confusing composition and aggregation**, e.g. modeling `ParkingSpot`s as something that could exist and be shared across multiple `ParkingLot`s when the problem clearly implies ownership. Minor, but a sharp interviewer notices.

## Pattern mistakes

- **Pattern-name-dropping without fit.** Saying "I'll use a Factory here" for a plain `new SomeClass()` with no variability, or wrapping everything in a Singleton "just in case." Overuse is graded down as hard as underuse (axis 4 in the evaluation framework) — only reach for a pattern when you can name the specific variability it's absorbing.
- **Singleton for state that doesn't need global enforcement.** Most "there's only one of these" requirements just mean "the composition root creates one instance" — they don't require the *class itself* to prevent multiple instantiation. Default to plain instantiation + dependency injection; only reach for real Singleton enforcement when the problem explicitly requires it (see the nuance in [patterns/01-creational-patterns.md](patterns/01-creational-patterns.md)).
- **Hardcoding a concrete dependency inside a business-logic class** (`new CreditCardPayment()` inside `ParkingLot`) instead of constructor-injecting an interface. This is a [DIP](02-solid-principles.md#d--dependency-inversion-principle) violation and it's exactly what breaks the "now swap in X" follow-up.
- **Reaching for Composite/Visitor/Flyweight when the problem doesn't call for them.** These are lower-frequency patterns — don't force them in to look sophisticated.

## Language-idiom mistakes

### Java-specific
- Using `int`/`String` constants instead of an `enum` for a small fixed set of variants (see [03-java-oop-essentials.md §7](03-java-oop-essentials.md#7-enums-with-behavior--not-just-constants)).
- Overriding `equals()` without `hashCode()` (silently breaks `HashMap`/`HashSet`).
- Choosing `abstract class` vs `interface` carelessly — this is explicitly watched (see [03-java-oop-essentials.md §2](03-java-oop-essentials.md#2-abstract-class-vs-interface--the-decision-interviewers-watch-for)).
- Public mutable fields, or no `final` on fields that are conceptually immutable after construction.
- Raw types / not using generics (`List` instead of `List<Vehicle>`).
- Writing every dependency as `new X()` at point of use instead of via constructor injection.

### Python-specific
- Writing Java in Python syntax: manual `get_x()`/`set_x()` methods everywhere instead of `@property`, or hand-rolled `__init__` + boilerplate `__eq__`/`__repr__` where `@dataclass` would do.
- A base class with `pass` bodies and a comment instead of `abc.ABC` + `@abstractmethod` — doesn't actually enforce the contract.
- Mutable default arguments (`def __init__(self, items=[])`) — a classic, very visible bug. Always `field(default_factory=list)` or `items=None` + `self.items = items or []`.
- No type hints on public method signatures — reads as less rigorous in an interview context even though Python doesn't require them.
- Ignoring the GIL/threading question entirely, or claiming Python has no concurrency concerns because of it (see [06-concurrency-essentials.md](06-concurrency-essentials.md) — the GIL still requires explicit locks for multi-step state changes).

## The meta-mistake

**Treating this as a coding-only round.** LLD is a *design communication* round that happens to end in code. A merely-correct class hierarchy delivered silently scores lower than a slightly simpler design where you clearly narrated trade-offs, invited the follow-up, and showed you know exactly why each choice was made. Optimize for legible reasoning, not maximal cleverness.

## Continue

Next: [08-final-checklist.md](08-final-checklist.md) — the pre-interview drill.
