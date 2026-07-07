# LLD (Low-Level Design) Cheatsheet

Quick reference for object-oriented design interviews. Source: algomaster.io/learn/lld — this is a condensed companion, not a replacement.

---

## Interview Approach (follow this order every time)

1. **Clarify requirements** — what are the core use cases? What's explicitly out of scope? Ask before designing.
2. **Identify entities/objects (nouns)** — the core classes in the system.
3. **Identify actions (verbs)** — these become methods on your classes.
4. **Define relationships** — inheritance (is-a), composition (has-a, owns lifecycle), aggregation (has-a, independent lifecycle), association (uses).
5. **Apply SOLID** — check your design against each principle out loud.
6. **Design classes/interfaces** — write out class signatures, key methods, important fields.
7. **Walk through use cases** — trace how a real scenario flows through your classes.
8. **Discuss extensibility & edge cases** — "if we needed to add X, here's what would change."

---

## SOLID Principles

| Principle | One-liner | Interview example |
|---|---|---|
| **S**ingle Responsibility | A class should have one reason to change | Split a `User` class that handles auth AND profile data AND notifications into three classes |
| **O**pen/Closed | Open for extension, closed for modification | Use a `PaymentStrategy` interface instead of an `if/else` chain over payment types — add new types without touching existing code |
| **L**iskov Substitution | Subtypes must be substitutable for their base type | A `Square extends Rectangle` breaks LSP if setting width also changes height unexpectedly |
| **I**nterface Segregation | Don't force classes to implement methods they don't need | Split a fat `Worker` interface into `Workable` and `Eatable` so a `Robot` doesn't need to implement `eat()` |
| **D**ependency Inversion | Depend on abstractions, not concrete implementations | A `NotificationService` should depend on a `MessageSender` interface, not directly on `EmailSender` |

---

## Design Patterns Quick Table

### Creational (object creation)
| Pattern | When to use |
|---|---|
| Singleton | Exactly one instance needed (config, connection pool). Watch for: makes testing harder, can hide dependencies. |
| Factory Method | Defer object creation to subclasses when the exact type isn't known until runtime |
| Abstract Factory | Create families of related objects without specifying concrete classes |
| Builder | Construct complex objects step-by-step (many optional parameters) |
| Prototype | Clone existing objects instead of creating from scratch (expensive initialization) |

### Structural (object composition)
| Pattern | When to use |
|---|---|
| Adapter | Make an incompatible interface work with your code (wrapping a third-party API) |
| Decorator | Add behavior to an object dynamically without subclassing (e.g. coffee with add-ons) |
| Facade | Provide a simple interface over a complex subsystem |
| Proxy | Control access to an object (lazy loading, access control, caching) |
| Composite | Treat individual objects and compositions uniformly (file/folder trees) |

### Behavioral (object interaction)
| Pattern | When to use |
|---|---|
| Observer | One-to-many notification (pub/sub within a single process — e.g. UI updates on state change) |
| Strategy | Swap an algorithm at runtime (different sorting/pricing/routing strategies behind one interface) |
| State | Object behavior changes based on internal state (order status: pending → shipped → delivered) |
| Command | Encapsulate a request as an object (undo/redo, task queues) |
| Iterator | Traverse a collection without exposing its internal structure |
| Chain of Responsibility | Pass a request along a chain of handlers until one handles it (middleware, approval workflows) |

---

## UML Quick Reference

- **Association** (uses): plain line — `Driver` uses `Car`
- **Aggregation** (has-a, independent lifecycle): line with hollow diamond — `Department` has `Professor`s, but professors exist independently
- **Composition** (has-a, owned lifecycle): line with filled diamond — `House` has `Room`s; rooms don't exist without the house
- **Inheritance** (is-a): line with hollow triangle arrow — `Car` is-a `Vehicle`
- **Realization** (implements): dashed line with hollow triangle — `Car` implements `Drivable`

---

## Common OOD Problems — Core Entities to Get Right

- **Parking Lot**: `ParkingLot`, `Level`, `ParkingSpot` (with `SpotType` enum: compact/large/handicap), `Vehicle` (with subtypes), `Ticket`, `PaymentProcessor`. Key discussion: spot assignment strategy, pricing strategy (Strategy pattern).
- **Elevator System**: `Elevator`, `ElevatorController`, `Request` (internal vs external, up vs down), `Floor`, `Door`. Key discussion: scheduling algorithm (nearest-car, SCAN/look algorithm), handling multiple elevators.
- **Library Management**: `Book`, `BookItem` (physical copy), `Member`, `Librarian`, `Loan`, `Reservation`, `Catalog` (search). Key discussion: search indexing, fine calculation, reservation queueing.
- **Vending Machine**: State pattern is the star here — `IdleState`, `HasMoneyState`, `DispensingState`, `OutOfStockState`. `Inventory`, `Product`, `PaymentProcessor`.
- **Chess / Tic-Tac-Toe**: `Board`, `Piece` (with subtypes for chess), `Player`, `Move`, `GameState`. Key discussion: move validation, check/checkmate detection (chess), win condition checking.
- **Rate Limiter as a class hierarchy**: `RateLimiter` interface with implementations `TokenBucketLimiter`, `SlidingWindowLimiter`, etc. — same algorithms as the HLD cheatsheet, but expressed as a class design with a common interface (Strategy pattern).
- **LRU Cache**: `LRUCache` backed by a `HashMap` + doubly linked list for O(1) get/put and O(1) eviction of least-recently-used. Classic interview question that bridges DSA and LLD.
- **Ride-Sharing Matching**: `Rider`, `Driver`, `Trip`, `MatchingService` (geospatial index + matching algorithm), `PricingStrategy` (surge pricing via Strategy pattern).

---

## API / Interface Design Notes

- Favor small, focused interfaces (Interface Segregation) over one large interface.
- Design for idempotency where retries are possible (e.g. `createOrder(idempotencyKey)`).
- Version your interfaces explicitly if extensibility is discussed (`PaymentProcessorV2`).
- Prefer composition over inheritance when behavior needs to be mixed and matched (e.g. a `FlyingCar` — don't force it into a rigid `Vehicle` → `Car`/`Plane` hierarchy).

---

## Common Pitfalls

- Jumping straight to code before identifying entities and relationships.
- Over-engineering with patterns that aren't needed — a pattern should solve a stated problem, not show off vocabulary.
- Ignoring SOLID until asked — mention it proactively as you design, not just when prompted.
- Not handling edge cases (what happens when the parking lot is full? when two elevator requests arrive simultaneously?).
- Designing classes with no clear single responsibility — usually a sign the entity list from step 2 needs revisiting.
