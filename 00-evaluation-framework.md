# The LLD Evaluation Framework

Know the rubric before you study to it. Every LLD interviewer — FAANG or startup — is silently scoring you on a variant of this. Nothing below is exotic; it's the shared structure behind "system design (low-level)" rounds.

## The 6 axes interviewers score

1. **Requirement gathering & scoping** — Did you ask clarifying questions before designing, or did you dive into code? Did you explicitly state assumptions and in/out-of-scope items?
2. **Object modeling** — Did you identify the right entities, the right relationships between them (is-a vs has-a vs uses-a), and avoid a God Object?
3. **SOLID adherence** — Not "did you name-drop SOLID" but "does your class hierarchy actually let you add a new payment type / vehicle type / notification channel without editing existing classes."
4. **Design pattern fit** — Did you reach for a pattern *because the problem called for it*, or did you cargo-cult Singleton/Factory everywhere? Overuse is graded down as hard as underuse.
5. **Code quality & language idiom** — Compiles-in-your-head correctness, proper access modifiers, correct use of interfaces/abstract classes, `equals`/`hashCode` where needed (Java) or dunder methods (Python), naming, and *not* writing pseudocode when asked for code.
6. **Extensibility under a follow-up** — The interviewer *will* say "now add X." Your design either absorbs it with one new class, or you're rewriting. This is the single biggest differentiator between pass/fail at senior level.

## The time budget (45-60 min interview)

| Phase | Time | What's scored |
|---|---|---|
| Clarify requirements | 5-8 min | Axis 1 |
| Identify core entities + relationships (talk + rough diagram) | 8-10 min | Axis 2 |
| Define interfaces/abstract classes, pick patterns | 5-7 min | Axes 3, 4 |
| Write core class code | 15-20 min | Axis 5 |
| Walk through 1-2 use cases end to end | 5 min | Axis 2, 5 |
| Handle "now add X" follow-up | 5-10 min | Axis 6 |

**Failure mode to avoid:** spending 25 minutes gold-plating requirements and diagrams, leaving 10 minutes to write code. Interviewers weight *working, extensible code* more than a beautiful diagram. Timebox the talk phase hard.

## What "senior" looks like vs "mid" vs "fails"

- **Fails**: one giant class with `if/else` chains on a `type` field; no interfaces; can't extend without editing 3 places; doesn't ask a single clarifying question.
- **Mid**: correct entities, uses inheritance appropriately, one or two patterns applied correctly, code compiles conceptually, but misses edge cases (concurrency, invalid input) unless prompted.
- **Senior**: proactively scopes ("are we handling concurrent bookings? I'll assume single-threaded first and mention how I'd extend for thread-safety"), models with interfaces from the start, picks the *minimal* pattern that solves the actual variability in the problem, anticipates the obvious follow-up before being asked, and narrates trade-offs ("I could use inheritance here but composition avoids the fragile base class problem because...").

## The narration habit

Score axis 3-6 are all easier to hit if you **talk while you design**: "I'm making `PaymentStrategy` an interface, not an abstract class, because there's no shared state or default behavior to inherit — this is pure behavioral substitution, so [Strategy pattern](patterns/03-behavioral-patterns.md) fits and it satisfies [Open/Closed](02-solid-principles.md#o--openclosed-principle)." This sentence alone hits 3 scoring axes in one breath. Practice saying things like this out loud, not just writing correct code silently.

## Language-specific scoring nuance

- **In Java**, interviewers notice: correct use of `interface` vs `abstract class`, `final` for immutability, proper `enum` usage (not `int` constants), overriding `equals`/`hashCode`/`toString` together, using the Collections framework idiomatically (`List<Vehicle>` not raw arrays), and not writing C-style code in Java syntax.
- **In Python**, interviewers notice: use of `abc.ABC` for real interfaces (not just convention), `@dataclass` for value objects, type hints, `Enum`, `@property`, and *not* writing Java in Python syntax (e.g., getters/setters everywhere instead of properties, or manual `__init__` boilerplate where a dataclass would do).

This asymmetry is exactly why [03-java-oop-essentials.md](03-java-oop-essentials.md) and [04-python-oop-essentials.md](04-python-oop-essentials.md) exist as separate, idiom-focused files rather than one generic "OOP" doc — the interviewer is partly evaluating whether you *actually know the language* vs transliterating from another one.

## Continue

Next: [01-requirements-and-uml.md](01-requirements-and-uml.md)
