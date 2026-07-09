# LLD Interview Knowledge Base — Java + Python

A self-contained, linked knowledge base for cracking Low-Level Design (LLD) interviews using **Java** and **Python** distinctly and well. Built for someone who is a strong, fast learner (10-12 yrs coding, strong in Python/Go/C++ OOP) but weak specifically in **Java idiom** — so Java gets more weight throughout.

Everything is local markdown. No web browsing needed — just follow the links.

## How this is organized

- **Core spine (root files)** — language-agnostic LLD skills: how you're evaluated, requirements/UML, SOLID, then Java and Python OOP idiom, then a cheatsheet, concurrency, mistakes, checklist.
- **`patterns/`** — the design patterns that actually show up in LLD interviews, with Java + Python code for each.
- **`problems/`** — the canonical LLD problem set. Each file: requirements → entities/class diagram → patterns used → Java implementation → Python implementation → follow-ups an interviewer will throw at you.

Every problem/pattern file cross-links back to the SOLID/pattern concept it exercises, so once the spine is loaded in your head, the problems reinforce it instead of teaching from scratch.

## The 5-hour critical path (packed)

This is the **priority-ordered** route through the material — it is *not* everything in this directory, it's the highest-signal subset. Everything else in `problems/` and `patterns/` is bonus depth (see "Beyond the 5 hours" below) — read it after, or skim it in spare minutes; it's here so you have a full week's worth of reference material on tap, not just a 5-hour script.

Timings assume you read at your own fast pace and *type out the code samples once* rather than just eyeballing them — typing is what makes Java syntax stick.

### Hour 1 — Foundations & mental model (0:00–1:00)
| Time | File | Why first |
|---|---|---|
| 10 min | [00-evaluation-framework.md](00-evaluation-framework.md) | Know the rubric before you study to it |
| 15 min | [01-requirements-and-uml.md](01-requirements-and-uml.md) | The first 5 min of every real interview |
| 35 min | [02-solid-principles.md](02-solid-principles.md) | The single highest-leverage topic — everything else is an application of this |

### Hour 2 — Language proficiency sprint (1:00–2:00)
| Time | File | Why |
|---|---|---|
| 35 min | [03-java-oop-essentials.md](03-java-oop-essentials.md) | Your weak spot — heaviest weighting here on purpose |
| 15 min | [04-python-oop-essentials.md](04-python-oop-essentials.md) | Refresher + the ABC/protocol/dataclass idiom interviewers expect |
| 10 min | [05-java-vs-python-cheatsheet.md](05-java-vs-python-cheatsheet.md) | Side-by-side so you stop context-switching mid-interview |

### Hour 3 — Design patterns deep dive (2:00–3:00)
| Time | File | Why |
|---|---|---|
| 5 min | [patterns/00-overview.md](patterns/00-overview.md) | Map of which pattern solves which smell |
| 20 min | [patterns/01-creational-patterns.md](patterns/01-creational-patterns.md) | Factory/Builder/Singleton show up constantly |
| 20 min | [patterns/02-structural-patterns.md](patterns/02-structural-patterns.md) | Decorator/Adapter/Composite/Facade/Proxy |
| 15 min | [patterns/03-behavioral-patterns.md](patterns/03-behavioral-patterns.md) | Prioritize Strategy, Observer, State, Command, Chain of Responsibility — skim the rest |

### Hour 4 — Practice problems, batch 1 — the "must-know" four (3:00–4:00)
| Time | File | Why this problem |
|---|---|---|
| 5 min | [problems/00-approach-framework.md](problems/00-approach-framework.md) | The repeatable 6-step script you run in *every* problem below |
| 15 min | [problems/01-parking-lot.md](problems/01-parking-lot.md) | THE canonical LLD problem — Strategy + Factory |
| 15 min | [problems/02-elevator-system.md](problems/02-elevator-system.md) | State machine + concurrency angle |
| 10 min | [problems/05-lru-cache-and-rate-limiter.md](problems/05-lru-cache-and-rate-limiter.md) | Data-structure-heavy LLD, very common as a warm-up problem |
| 15 min | [problems/06-splitwise-expense-sharing.md](problems/06-splitwise-expense-sharing.md) | Graph/ledger modeling, Strategy for split types |

### Hour 5 — Concurrency, batch 2, and close-out (4:00–5:00)
| Time | File | Why |
|---|---|---|
| 15 min | [06-concurrency-essentials.md](06-concurrency-essentials.md) | Elevator/parking-lot/booking follow-ups always probe thread-safety |
| 15 min | [problems/07-movie-ticket-booking.md](problems/07-movie-ticket-booking.md) | Concurrency-under-contention (seat locking) + Observer |
| 10 min | [problems/03-vending-machine.md](problems/03-vending-machine.md) | Clean State pattern showcase, fast to internalize |
| 10 min | [07-common-mistakes.md](07-common-mistakes.md) | What actually loses points — read this right before any interview |
| 10 min | [08-final-checklist.md](08-final-checklist.md) | Pre-interview 10-minute drill |

**End of 5 hours.** At this point you can competently run a full LLD interview in either language, including the two most commonly tested cross-cutting concerns (concurrency, extensibility).

## Beyond the 5 hours (the rest of the week's material)

These are fully written, same depth/format as above — read opportunistically, or the night before if you have a specific problem you're worried about:

- [problems/04-tic-tac-toe-and-chess.md](problems/04-tic-tac-toe-and-chess.md) — board-game modeling, good Strategy/Composite practice
- [problems/08-library-management-system.md](problems/08-library-management-system.md) — classic "entity relationship" heavy problem
- [problems/09-logging-framework.md](problems/09-logging-framework.md) — Chain of Responsibility + Builder in a systems-y context
- [problems/10-notification-and-observer-system.md](problems/10-notification-and-observer-system.md) — pure Observer/Strategy combo, pub-sub framing

## Quick navigation by concern

- **"I only have 20 minutes before the interview"** → [08-final-checklist.md](08-final-checklist.md) + [07-common-mistakes.md](07-common-mistakes.md)
- **"I keep freezing on Java syntax"** → [03-java-oop-essentials.md](03-java-oop-essentials.md) + [05-java-vs-python-cheatsheet.md](05-java-vs-python-cheatsheet.md)
- **"Interviewer asked about thread safety"** → [06-concurrency-essentials.md](06-concurrency-essentials.md)
- **"I don't know which pattern to reach for"** → [patterns/00-overview.md](patterns/00-overview.md)
- **"How do I even start a problem"** → [problems/00-approach-framework.md](problems/00-approach-framework.md)
