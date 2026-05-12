# Domain-Isolated Modular Driven Architecture (DIMDAh) — Domain Tier

## Preface

Modern software systems have grown increasingly complex while team sizes often remain small. With the rise of intelligent coding assistants and AI agents, developers can now build systems of unprecedented scale and sophistication — but the cognitive load of understanding and safely evolving such systems has become the new bottleneck. The **Domain-Isolated Modular Driven Architecture (DIMDAh)** aims to minimize this cognitive burden by introducing clear, enforceable boundaries between logical units of the system, allowing developers to modify, extend, or reason about a part of the system without understanding the entire codebase.

### The Cognitive Load Problem

In our view, the difference between a good architecture and a bad one is measured by **how much of the codebase a developer must read and understand to safely change a small part without fear of introducing new bugs**. Traditional architectures often suffer from:

- **Global dependencies**: Changing one module requires understanding dozens of others
- **Hidden coupling**: Side effects and implicit dependencies make changes risky
- **Unclear boundaries**: No obvious place where functionality belongs
- **Fragile abstractions**: Changes ripple unpredictably across the system

A well-designed architecture minimizes the "blast radius" of comprehension — when fixing a bug or adding a feature, you should only need to understand the immediate context, not the entire system.

### The AI Collaboration Challenge

Despite the impressive performance of modern AI coding assistants (such as Claude, GitHub Copilot, and others) in delivering code, they exhibit a critical limitation: **the larger and more interconnected a codebase, the harder it becomes to maintain coherence**. This isn't just about token limits — it's about reasoning across complex dependency graphs. When everything depends on everything else:

- AI agents struggle to reason about side effects and implicit coupling
- Context windows fill with tangentially related code, diluting focus
- Code suggestions become less precise and more error-prone
- The cognitive burden shifts from human to machine, but doesn't disappear

DIMDAh is designed to be AI-friendly: clear architectural boundaries mean AI agents can understand and modify isolated units effectively, just like human developers.

### The DIMDAh Approach

DIMDAh enforces separation of concerns through distinct layers, explicit contracts, and strong typing. It favors compile-time safety and logical isolation over runtime convenience, ensuring that logical errors are structurally impossible or at least clearly surfaced at development time.

## Architectural Goals

1. **Isolation by comprehension** — In order to mutate a part of logic, developers must fully understand only the unit they are modifying, _but not the entire system_.
2. **Clear structure** — It should always be obvious where a piece of logic or functionality belongs.
3. **Encapsulation** — Prevent invalid access to lower layers or unrelated components via enforced boundaries, to ensure logical consistency.
4. **No duplication of logic** — Prevent parallel or duplicated domain reasoning through explicit responsibilities.
5. **Static guarantees** — Encode system rules in the type system, ensuring that logical errors are caught at compile time as much as possible.
6. **Reduced mental load** — The design must prioritize clarity and reasoning simplicity over incidental flexibility.

---

## Architecture Overview

DIMDAh divides a system into three tiers:

| Tier | Responsibility |
|------|---------------|
| **Infrastructure** | Cross-cutting abstractions: logging, metrics, security, tracing. No business logic. |
| **Domain** | All business logic, organized into contexts with strict encapsulation. **This document.** |
| **Interface** | External-facing APIs and protocol handlers. Translates between protocols and domain. |

These tiers represent a deployment-agnostic logical separation. The Domain tier contains no knowledge of how it is exposed (Interface) or how cross-cutting concerns are implemented (Infrastructure).

> See the Infrastructure Tier and Interface Tier documents for their respective specifications.

---

## The Domain Tier

The Domain tier is the heart of the system. It contains all business logic and is organized according to the **Strict Encapsulation Tree (SET)** principle.

### Strict Encapsulation Tree (SET)

The codebase is organized as a tree of packages, where each node represents a unit of business logic. The fundamental rule of SET is:

> **A component can only be accessed by its direct parent in the tree.**

This means:
- Access flows strictly upward through the tree
- No sibling access (two lobes cannot call each other directly)
- No skip-level access (orch cannot reach into kernel directly, bypassing lobe)
- The one exception: **Port** and **Gate** nodes are designed to be accessed across context boundaries — Port (query interface) by any Lobe or Orch of another context; Gate (transaction interface) by Orchs of other contexts only

The package hierarchy directly models this tree, and **package-private visibility modifiers** allow the compiler to enforce these rules structurally, not just by convention.

### Context

A *context* is a bounded unit of domain responsibility. The Domain tier is composed of one or more contexts. Each context has four layers — Repo, Kernel, Lobe, and Orchestrator — plus a **Port** (query interface) and a **Gate** (transaction interface) that together form its public surface.

The presence and grouping of these layers within a context form a **story**: one can navigate the package tree to understand which kernels a lobe owns, which repos a kernel manages, and what the context exposes to the outside world.

---

## The Four Layers

### Repo

**Responsibility**: Communicate with the storage system (database, cache, message store). Define and manage DB-level data models.

**Access rules**:
- Can interact with the database
- **Cannot** contain business logic — only queries and persistence operations
- Can **only** be accessed by its owner Kernel (package-private to the kernel's package)

**Key principle**: DB-level models live here and adhere to constraints imposed by the storage technology (ORM annotations, column types, index hints). These models must never be exposed outside the Repo. The Kernel is solely responsible for translating between DB models and domain models.

```scala
// XRepo.scala — private to the kernels/x package
private[x] class XRepo(db: Database) {
  def findById(id: Long): IO[Nothing, Option[XRow]] = ...
  def insert(row: XRow): IO[DbError, Long] = ...
}

// DB-level model — never exposed outside this package
private[x] case class XRow(id: Long, email: String, status: String, createdAt: Instant)
```

### Kernel

**Responsibility**: Maintain the internal consistency of its domain aggregate, independently of the outer system.

**Access rules**:
- Can **only** access its own Repos
- **Cannot** communicate with any other component in the system (no other kernels, no lobes, no external ports)
- Can only be accessed by its owner Lobe (package-private to the lobe's package), or by Orch when Lobe is collapsed

**Key principle**: A Kernel defines the invariants of an aggregate. It translates between DB models and domain models, validates inputs, and enforces business rules that require no external context. A kernel should be understandable in complete isolation.

```scala
// XKernel.scala — private to the lobes/x package
private[x] class XKernel(repo: XRepo) {

  def register(email: String, hashedPw: String): IO[UserError, UserId] =
    for {
      existing <- repo.findByEmail(email)
      _        <- ZIO.when(existing.isDefined)(ZIO.fail(EmailAlreadyTaken))
      id       <- repo.insert(XRow(newId(), email, hashedPw, "active", now()))
    } yield UserId(id)

  def deactivate(id: UserId): IO[UserError, Unit] =
    repo.updateStatus(id.value, "inactive")
}
```

### Lobe

**Responsibility**: Decision-making and the majority of business rules. A Lobe is where complex orchestration *within* a context happens, coordinating kernels and incorporating information from outside the context.

**Access rules**:
- Can talk to its own Kernels
- Can query **other contexts through their Ports** (for reading data to inform decisions)
- **Cannot** directly access Repos
- **Cannot** call another context's Gate — cross-context mutations are Orch's responsibility
- Can only be accessed by its context's Orch

**Key principle**: A Lobe is where the *why* of business logic lives. It answers: "given the state of my kernels and the state of the world, what should happen?" It coordinates kernels to perform mutations and uses external Ports to gather the information it needs. It never triggers mutations in another context directly.

```scala
// XLobe.scala — private to the context package (accessible only by Orch)
private[context] class XLobe(kernel: XKernel, billingPort: BillingPort) {

  def registerWithPlan(
    email: String, pw: String, planId: PlanId
  ): IO[RegistrationError, UserId] =
    for {
      plan   <- billingPort.getPlan(planId)              // query outside context via Port
      _      <- ZIO.when(!plan.isActive)(ZIO.fail(PlanNotAvailable))
      userId <- kernel.register(email, hash(pw))         // delegate mutation to kernel
    } yield userId
}
```

### Orchestrator (Orch), Port, and Gate

A context exposes two public interfaces to the outside world:

**Gate** is the **transaction interface**. It defines all operations that constitute a business transaction within the context's domain. Foreign context Lobes may not access Gate — only Orchs of other contexts (or the Interface tier) may call Gate methods.

**Port** is the **query interface**. It exposes read-only operations that other contexts use to gather information. Both Lobes and Orchs of other contexts may access Port.

**Orch** implements both Gate and Port and has two responsibilities:
1. **API surface**: Delegate Gate and Port method calls to the appropriate Lobes
2. **Cross-context transactions**: Coordinate operations across multiple contexts — e.g., compensating transactions, sagas — when a single business operation spans more than one context

**Access rules**:
- Orch implements both Gate and Port (both are public)
- Orch can call Lobes within its context
- Orch **cannot** call Kernels or Repos directly (this would bypass lobe-level business rules)
- Orch can call other contexts' Gate and Port for cross-context coordination
- Lobe can query other contexts' **Port** (read-only); Lobe **cannot** call another context's Gate

```scala
// Port.scala — public (query interface)
trait UserPort {
  def getProfile(userId: UserId): IO[UserNotFound, UserProfile]
}

// Gate.scala — public (transaction interface; foreign Lobes may not access)
trait UserGate {
  def registerWithPlan(email: String, pw: String, planId: PlanId): IO[RegistrationError, UserId]
  def cancelPendingCheckout(userId: UserId): IO[UserNotFound, Unit]
}

// Orch.scala — public (implements both Port and Gate)
class UserOrch(lobe: UserLobe) extends UserPort with UserGate {

  def getProfile(userId: UserId) =
    lobe.getProfile(userId)

  def registerWithPlan(email: String, pw: String, planId: PlanId) =
    lobe.registerWithPlan(email, pw, planId)

  def cancelPendingCheckout(userId: UserId) =
    lobe.cancelPendingCheckout(userId)
}
```

For cross-context transactions, a higher-level Orch coordinates across Gates and Ports:

```scala
// CheckoutOrch — coordinates across context boundaries via Gate (mutations) and Port (reads)
class CheckoutOrch(userPort: UserPort, userGate: UserGate, orderGate: OrderGate) extends CheckoutGate {

  def checkout(userId: UserId, items: List[Item]): IO[CheckoutError, OrderId] =
    for {
      user    <- userPort.getProfile(userId)           // query via Port
      orderId <- orderGate.placeOrder(user, items)     // mutate via Gate
        .onError(_ => userGate.cancelPendingCheckout(userId)) // compensating action via Gate
    } yield orderId
}
```

---

## Layer Collapsing (Preventing Boilerplate)

The strict four-layer model ensures clean separation, but it can introduce boilerplate when there is no meaningful Lobe-level logic. If all business logic lives in the Kernel, and the Lobe would merely forward calls, the Lobe can be **collapsed**: Orch may call the Kernel directly.

**When to collapse**: There is no lobe-level logic — no cross-kernel decisions, no queries to external contexts, no higher-order business rules. All logic is either at the kernel level or the orch level.

**When to separate**: As soon as Lobe-level logic emerges — cross-kernel coordination, external Port queries, business rules that span multiple kernel states — the Lobe must be introduced as a distinct layer. At that point, the Kernel's API becomes package-private to the Lobe, and Orch must call the Lobe instead of the Kernel.

This is not a shortcut — it is a deliberate signal. The absence of a Lobe communicates to readers: *"there is no higher-order business logic here; the kernel handles it atomically."* When a Lobe later appears, it announces that the context has grown in complexity and now requires explicit business orchestration.

**Key discipline**: If you find yourself adding a Lobe that only forwards Kernel calls, do not add it — keep the collapsed structure. If you find the collapsed Orch growing business logic that queries other contexts or coordinates multiple kernels, introduce the Lobe and sever the layers.

---

## Subcontexts

A context may declare one or more **subcontexts** — bounded units of domain responsibility that are intentionally hidden from the rest of the system and controlled entirely within their parent context.

All rules that apply to a regular context (Repo, Kernel, Lobe, Orch, Port, Gate, SET, layer collapsing) apply equally to a subcontext. The distinction lies solely in **access control**:

- A subcontext **cannot be accessed by foreign contexts**. Only its parent context and its sibling subcontexts may use it.
- A subcontext has both a **Port** (query interface) and a **Gate** (transaction interface).
- The subcontext's **Port** may be accessed by its parent Orch, its parent's Lobes, and the Orchs and Lobes of sibling subcontexts.
- The subcontext's **Gate** may be accessed only by its parent Orch and the Orchs of sibling subcontexts.

**When to use a subcontext**: When a bounded unit of domain logic needs to be hidden from the outer world, and its access should be governed exclusively by the parent context. This is not a default decomposition strategy — use it deliberately, when control over exposure is the explicit goal.

In Scala, subcontext visibility is enforced by scoping the subcontext's Port, Gate, and Orch to the parent context's package:

```scala
// sub-a/Port.scala — accessible only within the parent context package
private[parentContext] trait SubAPort {
  def getSummary(id: SubAId): IO[SubANotFound, SubASummary]
}

// sub-a/Gate.scala — accessible only within the parent context package
private[parentContext] trait SubAGate {
  def create(data: SubAData): IO[SubAError, SubAId]
}

// sub-a/Orch.scala — private to parent context
private[parentContext] class SubAOrch(lobe: SubALobe) extends SubAPort with SubAGate {
  def getSummary(id: SubAId) = lobe.getSummary(id)
  def create(data: SubAData) = lobe.create(data)
}
```

The parent context's Orch wires and controls the subcontext:

```scala
// parent-context/Orch.scala — public
class ParentOrch(
  lobe: ParentLobe,
  subAOrch: SubAOrch    // injected; not exposed outside parent context
) extends ParentPort with ParentGate {

  def doSomething(id: SubAId): IO[ParentError, Result] =
    for {
      summary <- subAOrch.getSummary(id)    // query subcontext via its Port
      result  <- lobe.process(summary)
    } yield result
}
```

---

## Error Handling

> See [`error-handling.md`](./error-handling.md) for the full specification.

The core contract: all business logic functions return `IO[E, A]` where `E` is an explicit domain error type (a sealed ADT) and technical exceptions travel a separate channel. Each layer translates errors from its children into its own vocabulary — Repo errors never leak through Kernel, Kernel errors never leak through the Port.

---

## Package Structure

The package layout directly encodes the Strict Encapsulation Tree. Each level of nesting corresponds to a layer, and package-private visibility at each level enforces the access rules structurally.

```
context/
  ├── Port.scala           ← public (query interface)
  ├── Gate.scala           ← public (transaction interface; foreign Lobes cannot access)
  ├── Orch.scala           ← public (implements Port and Gate)
  ├── lobes/
  │   ├── x/
  │   │   ├── XLobe.scala
  │   │   └── kernels/
  │   │       ├── XKernel.scala
  │   │       └── XRepo.scala
  │   └── y/
  │       ├── YLobe.scala
  │       └── kernels/
  │           ├── a/
  │           │   ├── AKernel.scala
  │           │   └── repos/
  │           │       ├── ARepo1.scala
  │           │       └── ARepo2.scala
  │           └── b/
  │               ├── BKernel.scala
  │               └── BRepo.scala
  └── subcontexts/         ← optional; only when hiding from the outer world
      └── sub-a/
          ├── Port.scala   ← private[context] (parent + sibling Lobes/Orchs only)
          ├── Gate.scala   ← private[context] (parent + sibling Orchs only)
          ├── Orch.scala   ← private[context]
          └── lobes/
              └── ...
```

### Visibility Enforcement in Scala

```scala
// XRepo.scala — only XKernel can access this
private[x] class XRepo(db: DB) { ... }

// XKernel.scala — only XLobe can access this
private[x] class XKernel(repo: XRepo) { ... }

// XLobe.scala — only Orch can access this (private to the lobes package or context package)
private[context] class XLobe(kernel: XKernel, externalPort: SomePort) { ... }

// Port.scala — public query interface of the context
trait Port { ... }

// Gate.scala — public transaction interface; foreign Lobes may not access
trait Gate { ... }

// Orch.scala — public (wired externally, implements Port and Gate)
class Orch(xLobe: XLobe, yLobe: YLobe) extends Port with Gate { ... }
```

The compiler enforces:
- Orch cannot instantiate or access `XKernel` or `XRepo` (package-private to `x`)
- `XLobe` cannot access `XRepo` (package-private to `x`; lobe is in `lobes/x`, repo is in `lobes/x/kernels`)
- External code can only depend on `Port`, `Gate`, and `Orch` via dependency injection
- The Gate/Port discipline (Lobe may not call Gate of foreign context) is a convention enforced by code review or architectural tests, not the compiler

### What SET Rules the Compiler Cannot Enforce

Some rules require code review or architectural tests:
- Preventing Lobe from calling an external Gate (it may only query via Port)
- Preventing Lobe from performing mutations through an external Port (it may only query)
- Ensuring Orch does not embed business logic (only delegation and transaction coordination)
- Maintaining context isolation when language module systems are weak

Use tools like **ArchUnit**, **dependency analyzers**, or custom linters to enforce these automatically where possible.

---

## Dependency Enforcement Strategy

**Principle**: Maximize compile-time guarantees; minimize reliance on convention.

### Encapsulation via Visibility Scopes

- **Port** is public — the query surface of a context; accessible by any foreign Lobe or Orch
- **Gate** is public — the transaction surface of a context; accessible only by foreign Orchs (and the Interface tier), not Lobes
- **Orch** is public — but only ever injected/wired, never directly instantiated by consumers
- **Lobe** is package-private — accessible only within the context package (by Orch)
- **Kernel** is package-private — accessible only within the lobe's package
- **Repo** is package-private — accessible only within the kernel's package

Every component exposes a well-defined API and is free to have private members. But its API can only be accessed by the rules above.

### The Benefits of SET

These rules ensure:
- **One can always predict where a piece of code should live**, which reduces both read and write cost
- **No one can accidentally bypass business rules** (e.g., Orch cannot update a field directly in Repo without going through Kernel validation and Lobe-level decision)
- **Each layer is in complete ignorance of upper layer constraints**, making it easier to read, understand, and develop independently

---

## Architectural Challenges and Guidelines

### 1. Testing Strategy

DIMDAh's isolation enables testing at multiple levels with clear boundaries.

#### Repo Testing (Integration Tests)

Repos abstract persistence. Test them against real or in-memory databases.

**Characteristics**: Real queries and constraints, no business logic to test, use testcontainers or in-memory DBs, roll back after each test.

```scala
test("repo saves and retrieves row") {
  val row = XRow(1L, "test@example.com", "active", now())
  repo.insert(row).unsafeRunSync()

  val retrieved = repo.findById(1L).unsafeRunSync()
  assert(retrieved.contains(row))
}
```

#### Kernel Testing (Unit Tests)

Kernels are the innermost logic layer and should be tested in complete isolation with mocked repos.

**Characteristics**: Mock repos, test invariant enforcement and domain model translation, fast, deterministic.

```scala
test("kernel rejects duplicate email") {
  val mockRepo = mock[XRepo]
  when(mockRepo.findByEmail("a@b.com")).thenReturn(IO.succeed(Some(existingRow)))

  val result = kernel.register("a@b.com", "hash").unsafeRunSync()
  assert(result == Left(EmailAlreadyTaken))
}
```

#### Lobe Testing (Integration Tests with Mocks)

Lobes coordinate kernels and external Ports. Test with mock Ports and either real or mock Kernels.

**Characteristics**: Mock external Ports, verify business rule coordination, test decision logic.

```scala
test("lobe rejects registration when plan is inactive") {
  val mockBilling = mock[BillingPort]
  when(mockBilling.getPlan(planId)).thenReturn(IO.succeed(InactivePlan))

  val result = lobe.registerWithPlan("a@b.com", "pw", planId).unsafeRunSync()
  assert(result == Left(PlanNotAvailable))
}
```

#### Orch Testing (Orchestration Tests)

Orch tests verify delegation and cross-context transaction coordination.

**Characteristics**: Mock Lobes (and external Ports for cross-context Orch), verify compensating actions are triggered on failure.

```scala
test("checkout orch compensates user on order failure") {
  val mockUser = mock[UserPort]
  val mockOrder = mock[OrderPort]
  when(mockOrder.placeOrder(any, any)).thenReturn(IO.fail(OrderFailed))

  checkoutOrch.checkout(userId, items).unsafeRunSync()

  verify(mockUser).cancelPendingCheckout(userId)
}
```

#### Testing Pyramid

DIMDAh naturally produces a healthy testing pyramid:

```
        /\
       /  \  Orch (few — orchestration and compensation paths)
      /____\
     /      \  Lobe (moderate — business rule combinations)
    /________\
   /          \  Kernel (many — invariant coverage)
  /____________\
 /              \  Repo (targeted — query correctness)
/________________\
```

#### Best Practices

1. **Test behavior, not implementation**: Focus on what the component does, not how
2. **Mock at architectural boundaries**: Mock Ports (external), mock Repos (for kernel tests), not internal helpers
3. **Leverage property-based testing**: For Kernels, use libraries like ScalaCheck to cover invariant space
4. **Use architectural tests**: Tools like ArchUnit verify SET dependency rules

### 2. Event-Driven Communication

**Problem**: SET's strict isolation prevents direct cross-context dependencies. But many operations require coordination — for example, when a user updates their profile, a read model in another context may need updating.

**Solution**: **Domain events** serve as the communication mechanism between contexts.

#### Event Emission

When a domain model is mutated, the responsible Kernel or Lobe **emits a domain event** rather than directly calling other contexts:

```scala
// Correct: emit event, let other contexts react
def updateProfile(userId: UserId, data: ProfileData): IO[Error, Unit] =
  for {
    _ <- kernel.applyProfileUpdate(userId, data)
    _ <- eventPublisher.emit(UserProfileUpdated(userId, data))
  } yield ()
```

Domain events are **facts** — immutable records that something happened. They represent state changes, not commands.

#### Event Consumption

Other contexts subscribe to relevant events and react:

```scala
eventBus.subscribe[UserProfileUpdated] { event =>
  readModelLobe.updateUserView(event.userId, event.data)
}
```

Consumers are independent — the emitting context has no knowledge of who is listening.

#### Implementation Options

- **Internal Event Bus**: Akka Event Stream, in-memory pub/sub — for same-process communication
- **External Message Queue**: Kafka, RabbitMQ, AWS SNS/SQS — for distributed systems

The choice is a deployment decision. Domain logic remains unchanged.

#### Transactional Outbox Pattern

To ensure events are published reliably alongside database changes:

```scala
def updateProfile(userId: UserId, data: ProfileData): IO[UpdateError, Unit] =
  transactionally {
    for {
      _ <- kernel.applyProfileUpdate(userId, data)
      _ <- outboxRepo.insert(OutboxEvent("UserProfileUpdated", UserProfileUpdated(userId, data)))
    } yield ()
  }
// A separate process polls the outbox and publishes events
```

#### Trade-offs

- **Eventual consistency**: Subscribers process events asynchronously
- **Idempotency**: Event handlers must be safely re-entrant
- **Event versioning**: Plan for schema evolution as the system grows

### 3. Transaction Boundaries and Consistency

Transactions respect isolation boundaries. The scope of a transaction determines what consistency guarantees you can provide.

#### Within a Kernel (Atomic per Repo Call)

Kernels coordinate their Repos. Where multiple repo operations must be atomic, use a single database transaction:

```scala
def transfer(from: AccountId, to: AccountId, amount: Money): IO[TransferError, Unit] =
  transactionally {
    for {
      _ <- repo.withdraw(from, amount)
      _ <- repo.deposit(to, amount)
    } yield ()
  }
```

#### Within a Lobe (Single Context Transaction)

A Lobe may coordinate multiple Kernels within a transaction if they share the same database:

```scala
def processPaymentAndActivate(userId: UserId, amount: Money): IO[PaymentError, Unit] =
  transactionally {
    for {
      _ <- paymentKernel.charge(userId, amount)
      _ <- subscriptionKernel.activate(userId)
    } yield ()
  }
```

#### Across Contexts (Orch-Managed — Saga Pattern)

Contexts have independent databases. Cross-context consistency requires the **Saga pattern** with compensating actions, managed by Orch:

```scala
def checkout(userId: UserId, items: List[Item]): IO[CheckoutError, OrderId] =
  for {
    _       <- inventoryPort.reserve(items)
                 .onError(_ => IO.unit) // nothing to compensate yet
    _       <- paymentPort.charge(userId, total)
                 .onError(_ => inventoryPort.release(items))
    orderId <- orderPort.create(userId, items)
                 .onError(_ =>
                   paymentPort.refund(userId, total) *>
                   inventoryPort.release(items)
                 )
  } yield orderId
```

#### Decision Matrix

| Scope | Pattern | Consistency |
|-------|---------|-------------|
| Single Kernel | DB transaction | ACID |
| Multiple Kernels in a Lobe (same DB) | DB transaction | ACID |
| Cross-context (separate DBs) | Events + eventual consistency | Eventual |
| Cross-context with coordination | Saga + compensating actions | Eventual |
| Reliable event delivery | Transactional outbox | Eventual + reliable |

### 4. Shared Model Strategy

**Core Principle**: Type duplication is acceptable; logic duplication is forbidden.

#### Internal vs Exposed Models

Each context maintains private internal models and exposes only specific views through its Port:

- **Internal models**: Persistence entities, intermediate computation state — never exposed outside the context
- **Port models**: Public data types returned or accepted by Port methods — form the contract with consumers

```scala
// Internal (package-private to context)
private[context] case class UserEntity(id: Long, email: String, pwHash: String, status: String)

// Exposed via Port (public)
case class UserProfile(userId: UserId, email: Email, status: UserStatus)
case class UserSummary(userId: UserId, email: Email)
```

#### Common Module for Shared Primitives

A **Common module** contains universally shared primitive types with no behavioral differences across contexts:

```scala
// Common module
case class UserId(value: Long) extends AnyVal
case class Email(value: String) extends AnyVal
case class Money(amount: BigDecimal, currency: Currency)
```

Not appropriate for Common: rich domain models, types with complex validation, types that may diverge across contexts.

#### When to Duplicate

Duplicate types freely when different contexts need different representations of the same concept, or when the types may evolve independently. **Type duplication is a feature, not a failure.** It preserves context autonomy.

### 5. Documentation Discipline

Every context must include a clear description of:
- Its purpose and the domain it encapsulates
- Its invariants and what it guarantees
- Its dependencies (which Ports it calls)

Documentation is as essential as code — for both developers and AI agents collaborating on the codebase.

### 6. Cross-Cutting Concerns

Cross-cutting concerns (logging, metrics, security, tracing) are provided by the **Infrastructure tier** and injected into Domain components. Domain components depend on Infrastructure *interfaces* only — never on concrete implementations.

By injecting these dependencies (constructor injection, Reader monad, ZIO environment), business logic remains:
- **Testable**: Mock implementations in tests
- **Pure**: No direct I/O or side effects in domain code
- **Portable**: Implementation details can change without affecting business logic

See the Infrastructure Tier document for interface definitions and conventions.

### 7. Performance Considerations

DIMDAh's isolation provides natural instrumentation points:

- **Measure at layer boundaries**: Trace latency across Repo, Kernel, Lobe, and Orch
- **Cache at boundaries**: Add caching decorators to Repos or Port implementations without polluting business logic
- **Batch in Repos**: Repository interfaces can expose batch operations while maintaining abstraction
- **Async event handling**: Non-critical events can be forked and processed asynchronously

**General advice**: Measure first. DIMDAh's clean boundaries make targeted optimization straightforward once you identify actual bottlenecks.

### 8. Runtime Deployment

The Domain tier's logical context boundaries are independent of deployment topology. A system can start as a monolith (all contexts in-process) and extract contexts into separate services without major refactoring, as long as architectural boundaries were respected. The Ports serve as natural service boundaries.

---

## Naming Conventions

Consistent naming makes the architecture self-documenting.

### Domain Models (Port-Exposed)

- Pattern: Singular noun representing the entity
- Examples: `UserProfile`, `OrderSummary`, `ProductView`

### Value Objects

- Pattern: `<Domain><Concept>` or just `<Concept>`
- Examples: `UserId`, `Email`, `Money`, `OrderStatus`

### Repo

- Pattern: `<Domain>Repo`
- Examples: `UserRepo`, `OrderRepo`, `ProductRepo`

### Kernel

- Pattern: `<Domain>Kernel`
- Examples: `UserKernel`, `OrderKernel`, `InventoryKernel`

### Lobe

- Pattern: `<Domain>Lobe`
- Examples: `UserLobe`, `CheckoutLobe`, `FulfillmentLobe`

### Port, Gate, and Orch

- Port (query interface): `<Context>Port` or just `Port` when inside the context package
- Gate (transaction interface): `<Context>Gate` or just `Gate` when inside the context package
- Orch: `<Context>Orch` or `Orch`
- Examples: `UserPort`, `UserGate`, `UserOrch`, `CheckoutPort`, `CheckoutGate`, `CheckoutOrch`

### Domain Errors

- Pattern: `<Context><ErrorCondition>`
- Use past tense or noun form
- Examples: `UserNotFound`, `InsufficientBalance`, `InvalidEmailFormat`, `EmailAlreadyTaken`

### Domain Events

- Pattern: `<Entity><ActionInPastTense>`
- Always past tense (facts that happened)
- Examples: `UserRegistered`, `OrderPlaced`, `PaymentCompleted`, `ProfileUpdated`

### Function Naming

**Repo functions**:
- CRUD: `findById`, `findByEmail`, `insert`, `update`, `delete`
- Query: `findBy<Criteria>`, `listBy<Criteria>`

**Kernel functions**:
- Business operations: `register`, `deactivate`, `applyDiscount`, `validateAndCreate`

**Lobe functions**:
- Business operations with intent: `registerWithPlan`, `checkoutItems`, `cancelOrder`

**Orch / Port functions**:
- Business capabilities: `getProfile`, `placeOrder`, `processPayment`

**Event handlers**:
- Pattern: `on<Event>` or `handle<Event>`
- Examples: `onUserRegistered`, `handleOrderPlaced`

### General Guidelines

1. **Be explicit over clever**: `UserRegistrationError` is better than `RegErr`
2. **Use domain language**: If the business calls it "fulfillment", don't call it "shipping"
3. **Avoid technical prefixes**: Don't prefix interfaces with `I`
4. **Package visibility matters**: If it's package-private, the name can be implementation-focused
5. **Consistency within context**: Once you choose a convention, stick with it

---

## Summary

The **Domain-Isolated Modular Driven Architecture (DIMDAh)** structures business logic into contexts governed by the **Strict Encapsulation Tree**. Access flows strictly upward through the tree; each layer knows only its children, not its parents or siblings.

### The Four Layers and Their Rules

| Layer | Can Access | Cannot Access | Accessed By |
|-------|-----------|---------------|-------------|
| **Repo** | Database | Anything else | Owner Kernel only |
| **Kernel** | Own Repos | Other Kernels, Lobes, Ports | Owner Lobe (or Orch if collapsed) |
| **Lobe** | Own Kernels; external **Ports** (read) | Repos; external Gates; cross-context mutations | Context Orch |
| **Orch** | Own Lobes; external Ports and Gates | Own Kernels, Repos | Interface tier; other Orchs |

**Port** (query) and **Gate** (transaction) are the two public surfaces of a context. All external access goes through one of them. Lobes may only call foreign **Port**s — never foreign **Gate**s.

### Core Tenets

1. **Cognitive load is the bottleneck** — Good architecture minimizes how much you must understand to make changes safely
2. **Compiler over convention** — Encode rules in package visibility, not just documentation
3. **Domain errors are data** — Make failure modes explicit in function signatures
4. **Type duplication is acceptable; logic duplication is forbidden** — Duplicate types to prevent coupling, never duplicate business logic
5. **Collapse layers when logic doesn't require them** — Avoid boilerplate, but restore layers the moment complexity demands it
6. **Events enable isolation** — Cross-context communication uses immutable events, not direct calls
7. **Transactions respect boundaries** — Transaction scope aligns with isolation boundaries; cross-context consistency uses Sagas

### When to Use DIMDAh

Well-suited for:
- Complex business domains with substantial business logic
- Long-lived systems that will evolve over years
- Small to medium teams that need to move quickly without breaking things
- AI-assisted development where clear boundaries help both humans and machines reason about code

May be overkill for:
- Simple CRUD applications with minimal business logic
- Prototypes and experiments with short lifespans
- Systems where performance constraints require tight coupling

### Adoption Steps

1. **Identify your contexts** — Map business capabilities to bounded contexts
2. **Define Ports and Gates first** — What queries does each context expose (Port)? What transactions does it expose (Gate)?
3. **Implement Kernels** — Start with pure invariant logic and repos
4. **Add Lobes when logic requires** — Introduce the layer when cross-kernel coordination or external queries are needed
5. **Wire via Orch** — Implement Ports, introduce cross-context transactions as requirements emerge
6. **Use package-private visibility** — Let the compiler enforce SET from day one

DIMDAh is a living architecture — adapt it to your context while preserving the core principle: **minimize cognitive load through enforced isolation**.

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                     Domain Tier                                 │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Context A                                               │   │
│  │                                                          │   │
│  │  ┌──────────┐ ┌──────────┐  ┌──────────────────────┐   │   │
│  │  │   Port   │ │   Gate   │◄─│         Orch         │   │   │
│  │  │ (query)  │ │  (txns)  │  │ (implements Port+Gate│   │   │
│  │  └──────────┘ └──────────┘  │  manages cross-ctx   │   │   │
│  │       ▲            ▲        │       txns)          │   │   │
│  │       │            │        └──────────┬───────────┘   │   │
│  │   any Lobe/    Orch only               │ calls         │   │
│  │   Orch only    (no foreign             │               │   │
│  │                 Lobes)                 │               │   │
│  │                  ┌─────────────────────▼───────────┐   │   │
│  │                  │           Lobe X                 │   │   │
│  │                  │  (decision making, business rules│   │   │
│  │                  │  queries other Ports only)       │   │   │
│  │                  └──────────┬──────────────────────┘   │   │
│  │                             │ calls                      │   │
│  │                  ┌──────────▼──────────────────────┐   │   │
│  │                  │          Kernel X                │   │   │
│  │                  │  (invariants, aggregate rules)   │   │   │
│  │                  └──────────┬──────────────────────┘   │   │
│  │                             │ calls                      │   │
│  │                  ┌──────────▼──────────────────────┐   │   │
│  │                  │           Repo X                 │   │   │
│  │                  │  (persistence, DB-level models)  │   │   │
│  │                  └──────────┬──────────────────────┘   │   │
│  │                             │                            │   │
│  └─────────────────────────────┼────────────────────────── ┘   │
│                                │ reads/writes                   │
│                       ┌────────▼────────┐                      │
│                       │    Database     │                      │
│                       └─────────────────┘                      │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Context B                                               │   │
│  │  ┌──────────┐ ┌──────────┐  ┌──────────────────────┐   │   │
│  │  │   Port   │ │   Gate   │◄─│         Orch         │   │   │
│  │  └──────────┘ └──────────┘  └──────────────────────┘   │   │
│  │       ▲                                 │               │   │
│  └───────┼─────────────────────────────────┼───────────────┘   │
│          │ Port.getPlan()                   │                   │
│          └──── Lobe X (Context A) queries ─┘                   │
│               Context B's Port (read-only)                     │
└─────────────────────────────────────────────────────────────────┘


Layer Access Rules (Strict Encapsulation Tree):
═══════════════════════════════════════════════

  Port ◄── any foreign Lobe or Orch (read/query only)
  Gate ◄── foreign Orchs only (no foreign Lobes)
    │
  Orch ──► Lobe  [cannot reach Kernel or Repo]
    │         │
    │      Kernel ──► Repo ──► Database
    │
  (for cross-ctx txns) ──► other Context's Gate
  (for cross-ctx reads) ──► other Context's Port

  Lobe may also: ──► other Context's Port (read-only, never Gate)

  Event Flow (Cross-Context):
  ═══════════════════════════

  Lobe/Kernel                           Another Context
      │                                       │
      │ 1. mutate state                       │
      │ 2. emit event ──► EventBus ──────────►│ subscribe
      │                                       │ 3. react (update read model,
      │                                       │    trigger workflow, etc.)
      │                                       │
      │  (emitter has no knowledge of         │
      │   subscribers or their reactions)     │
```

### Key Architectural Flows

**1. Request Processing (Top-Down)**:
```
Interface tier
  → Gate or Port (context boundary)
    → Orch (delegates, manages cross-context txns)
      → Lobe (business rules, coordinates kernels)
        → Kernel (invariant enforcement)
          → Repo (persistence)
            → Database
```

**2. Cross-Context Read (Lobe queries external Port)**:
```
Lobe X (Context A)
  → Context B's Port (read query — never Gate)
    → Context B's Orch
      → Context B's Lobe
        → Context B's Kernel
          → Context B's Repo
```

**3. Cross-Context Transaction (Orch-managed Saga)**:
```
CheckoutOrch
  → UserPort.getProfile()          [read via Port]
  → InventoryGate.reserve()        [mutate via Gate — step 1]
  → PaymentGate.charge()           [mutate via Gate — step 2; compensates step 1 on failure]
  → OrderGate.create()             [mutate via Gate — step 3; compensates steps 1 & 2 on failure]
```

**4. Collapsed Layers (no Lobe-level logic)**:
```
Orch ──► Kernel  [Lobe omitted — no higher-order business rules exist yet]
            └──► Repo
```
