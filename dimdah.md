# Domain-Isolated Modular Driver Architecture (DIMDAh)

## Preface

Modern software systems have grown increasingly complex while team sizes often remain small. With the rise of intelligent coding assistants and AI agents, developers can now build systems of unprecedented scale and sophistication — but the cognitive load of understanding and safely evolving such systems has become the new bottleneck. The **Domain-Isolated Modular Driver Architecture (DIMDAh)** aims to minimize this cognitive burden by introducing clear, enforceable boundaries between logical units of the system, allowing developers to modify, extend, or reason about a part of the system without understanding the entire codebase.

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

DIMDAh is designed to be AI-friendly: clear architectural boundaries mean AI agents can understand and modify isolated units effectively, just like human developers. By constraining the "blast radius" of each change, both humans and AI can work more confidently.

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

## Layer Overview

### 0. Infrastructure Layer

This is the **foundational layer** that all other layers may depend on. It provides common abstractions for cross-cutting concerns without containing any business logic.

#### Contains:

* **Logging abstractions** — Interfaces for structured logging
* **Metrics interfaces** — Counters, timers, gauges for observability
* **Security context definitions** — User identity, permissions, authentication tokens
* **Tracing utilities** — Correlation IDs, span management for distributed tracing
* **Common utilities** — Time providers, ID generators, configuration interfaces

#### Responsibilities:

* Provide **interfaces and abstractions only** — concrete implementations are injected at runtime
* Remain stable and domain-agnostic
* Enable testability through abstraction (mock loggers, test clocks, etc.)

#### Key Principle:

The Infrastructure layer contains **no implementations** of these abstractions — only interfaces/traits. This preserves the dependency rule: higher layers depend on abstractions, not concretions. Implementations are provided at the application boundary (composition root).

**Example**:
```scala
trait Logger[F[_]] {
  def info(msg: String, context: Map[String, String]): F[Unit]
  def error(msg: String, ex: Throwable): F[Unit]
}

trait Metrics[F[_]] {
  def increment(counter: String, tags: Map[String, String]): F[Unit]
  def time[A](operation: String)(fa: F[A]): F[A]
}
```

### 1. Core (Domain) Layer

This layer defines the most atomic, stable building blocks of the domain. It focuses on representing **business entities and invariant-preserving logic**.

#### Contains:

* **Domain Models** — Immutable, validated data structures.
* **Kernel Units** — Atomic logical units that encapsulate core rules.
* **Repositories** — Abstract interfaces for persistence.
* **DAOs** — Concrete persistence implementations.
* **Event Publishers** — For emitting domain-level events.

#### Responsibilities:

* Define the essence of the domain.
* Avoid dependencies on higher layers.
* Be purely functional and deterministic whenever possible.

#### Notes:

* A `Kernel` is the smallest isolated logical component in the domain.
* Each kernel should be testable and usable independently.

### 2. Subdomain Layer

This layer represents **bounded contexts** that combine multiple kernels to form a coherent logical unit. Each subdomain is self-contained and exposes a stable API to higher layers.

#### Contains:

* **Context Units** — Logical modules integrating multiple kernels.
* **Subdomain Events** — Typed events for inter-context communication.

#### Responsibilities:

* Combine related kernels to fulfill cohesive domain behaviors.
* Hide internal implementation from other subdomains.
* Expose a narrow, intention-revealing API.

### 3. Service (Orchestration) Layer

This layer coordinates multiple subdomains into business processes. It orchestrates logic flow without containing business rules itself.

**Critical distinction**: A **Service** represents a deployment boundary. Each service should be independently deployable, with its own persistence, development lifecycle, and operational concerns. If you were implementing this as microservices, each service would map to a separate microservice.

**Subdomains vs Services**:
- **Subdomains** are logically isolated but may be coupled. Multiple related subdomains reside together within a single service.
- **Services** are independently deployable units with no shared persistence or tight coupling to other services.

The key question: "Could these two components exist as separate microservices?" If yes → separate services. If no (too coupled) → subdomains within the same service.

#### Contains:

* **Logic Units (Processes)** — High-level workflows coordinating subdomain interactions within this service.
* **Error Composition** — Translating fine-grained errors between contexts.

#### Responsibilities:

* Model service-level processes.
* Integrate multiple subdomains within this service boundary.
* Manage transaction boundaries or compensations if applicable.
* Communicate with other services via well-defined APIs or events (loosely coupled).

#### Notes:

While this layer risks becoming a "kitchen sink," strict adherence to subdomain boundaries and strong typing keeps it under control. The architecture does not prescribe whether services are deployed as microservices or as modules within a monolith — that remains a deployment decision — but the logical boundaries must support independent deployment.

### 4. Interface Layer

This layer exposes the system to the outside world. It’s the boundary where technical concerns meet the domain model.

#### Contains:

* **Endpoints** — Public APIs (HTTP, gRPC, messaging, etc.).
* **DTOs / View Models** — External-facing data representations.
* **Protocol Handlers** — Input/output translation, authentication, etc.

#### Responsibilities:

* Translate between external protocols and internal domain representations.
* Delegate all real work to orchestration or subdomain layers.
* Never contain business logic.

---

## Error Handling Model

All logic functions return values in an effect type that separates **technical exceptions** from **domain errors**.

### Core Principle: Domain Errors Are Data

**Domain errors are not exceptions** — they are expected outcomes that represent business rule violations. As such, they must be explicitly declared in function signatures as part of the type contract.

**`IO[E, A]`** — Represents a computation that may:
- Fail with technical exceptions (infrastructure failures, I/O problems, unexpected runtime errors)
- Fail with domain errors of type `E` (business rule violations)
- Succeed with value `A`

Example signature:
```scala
def registerUser(data: UserData): IO[UserRegistrationError, UserId]
```

By encoding domain errors in the type signature, we make invalid states visible at compile time and force callers to handle all logical outcomes explicitly. This enforces clarity in error semantics and prevents logical ambiguity.

### Implementation Options

The `IO[E, A]` type is a semantic contract, not a prescribed implementation. Depending on your platform and language, you might implement it as:

- **ZIO** (Scala): `ZIO[Any, E, A]` — without the environment/resource layer
- **Custom wrapper**: A wrapper class over `Future[Either[E, A]]` with helper methods for composition
- **Cats Effect** (Scala): With explicit error handling layers
- **Arrow or similar** (Kotlin): Effect systems with typed errors
- **Language-native mechanisms**: Any approach that preserves the distinction between technical and domain errors

**Note**: This architecture is most naturally expressed in functional programming languages with advanced type systems (Scala, Haskell, F#, OCaml, Rust, etc.). Languages without strong static typing or effect type support will require more discipline and convention to maintain the architectural guarantees.

---

## Dependency Enforcement Strategy

**Principle**: Maximize compile-time guarantees; minimize reliance on convention.

The architecture prioritizes **compiler-enforced boundaries** over documentation or code review discipline. Where the compiler cannot help, conventions are documented and enforced through review.

### Encapsulation via Visibility Scopes

Layer boundaries and access restrictions are enforced using **package-private (or module-private) visibility modifiers**:

- **Public APIs** are explicitly marked and represent the contract exposed to higher layers
- **Internal implementations** (DAOs, internal models, helper utilities) are package-private and inaccessible from outside
- Higher layers interact only through public interfaces, with the compiler preventing direct access to implementation details

**Example**: In the Core (Domain) layer:
- `UserRepository` (interface) → **public** — accessible to Subdomain and Service layers
- `UserDAO` (concrete implementation) → **package-private** — only accessible within Core (Domain)
- Internal domain model constructors or validation logic → **package-private** — ensuring invariants are maintained

This explains why **domain models and persistence implementations coexist in the Core (Domain) layer**: they are packaged together so that visibility modifiers can enforce the abstraction boundary at compile time. Higher layers cannot bypass the repository interface to access DAOs directly.

### When Convention Is Required

Some architectural rules cannot be enforced by the compiler:
- Preventing business logic in the Interface layer
- Ensuring Services don't contain domain rules (only orchestration)
- Maintaining subdomain isolation (when language module systems are weak)

For these cases, **code reviews and architectural tests** (such as ArchUnit, dependency analyzers, or linters) provide secondary enforcement.

---

## Architectural Challenges and Guidelines

### 1. Testing Strategy

DIMDAh's isolation enables **testing at multiple levels** with clear boundaries. Each layer has distinct testing characteristics and requirements.

#### Kernel Testing (Unit Tests)

Kernels are **purely functional** and contain core business logic. They should be tested in complete isolation:

**Characteristics**:
- No I/O, no side effects
- Deterministic: same input always produces same output
- Dependencies (if any) are injected as parameters
- Fast execution (microseconds per test)

**Example**:
```scala
test("user registration validates email format") {
  val result = UserKernel.validateEmail("invalid-email")
  assert(result.isLeft)
  assert(result.left.get == InvalidEmailFormat)
}

test("order total calculation includes tax") {
  val items = List(OrderItem(Money(100), quantity = 2))
  val total = OrderKernel.calculateTotal(items, taxRate = 0.1)
  assert(total == Money(220)) // (100 * 2) * 1.1
}
```

**Approach**: Pure unit tests with no mocking needed.

#### Repository Testing (Integration Tests)

Repositories abstract persistence. Test them against real or in-memory databases:

**Characteristics**:
- Test actual persistence behavior
- Use in-memory databases (H2, SQLite) or testcontainers
- Verify queries, transactions, constraint handling
- Slower than unit tests but still fast (milliseconds)

**Example**:
```scala
test("user repository saves and retrieves user") {
  val user = User(UserId(1), Email("test@example.com"))
  userRepo.save(user).unsafeRunSync()

  val retrieved = userRepo.findById(UserId(1)).unsafeRunSync()
  assert(retrieved.contains(user))
}
```

**Approach**: Integration tests with test databases. Use transactions that roll back after each test.

#### Subdomain Testing (Integration Tests with Mocks)

Subdomains combine multiple kernels and depend on repositories. Test with **mock repositories** to isolate business logic:

**Characteristics**:
- Mock repository dependencies
- Test coordinated kernel interactions
- Verify event emission
- Medium speed (milliseconds per test)

**Example**:
```scala
test("user registration sends welcome email") {
  val mockRepo = mock[UserRepository]
  val mockEventPublisher = mock[EventPublisher]
  val subdomain = UserSubdomain(mockRepo, mockEventPublisher)

  subdomain.registerUser(userData).unsafeRunSync()

  verify(mockRepo).save(any[User])
  verify(mockEventPublisher).emit(UserRegistered(...))
}
```

**Approach**: Mock external dependencies, test business logic flows.

#### Service Testing (End-to-End Orchestration Tests)

Services orchestrate multiple subdomains. Test complete workflows:

**Characteristics**:
- Test cross-subdomain coordination
- May use mix of real and mock dependencies
- Verify error handling and compensation
- Slower (10s-100s of milliseconds)

**Example**:
```scala
test("complete order workflow") {
  // Use real subdomains with mocked repositories
  val result = orderService.placeOrder(customerId, items)

  assert(result.isRight)
  assert(inventoryRepo.wasCalledWith(items))
  assert(paymentRepo.wasCalledWith(customerId))
  assert(eventPublisher.emittedEvent[OrderPlaced])
}
```

**Approach**: Integration tests at service boundary, testing workflows end-to-end.

#### Interface Layer Testing (Contract Tests)

The Interface layer exposes APIs. Test protocol handling and DTO translation:

**Characteristics**:
- Test HTTP/gRPC endpoints
- Verify authentication, authorization
- Test DTO serialization/deserialization
- Mock service layer dependencies

**Example**:
```scala
test("POST /users returns 201 on success") {
  val mockService = mock[UserService]
  when(mockService.registerUser(any)).thenReturn(IO.pure(UserId(1)))

  val response = httpClient.post("/users", userJson)
  assert(response.status == 201)
  assert(response.header("Location") == "/users/1")
}
```

**Approach**: HTTP contract tests with mocked services.

#### Event Handler Testing

Event handlers react to domain events. Test in isolation:

**Characteristics**:
- Test each handler independently
- Mock dependencies (repositories, external services)
- Verify idempotency
- Test error handling and retries

**Example**:
```scala
test("user profile updated event updates read model") {
  val mockReadModelRepo = mock[ReadModelRepository]
  val event = UserProfileUpdated(UserId(1), "New Name")

  eventHandler.handle(event).unsafeRunSync()

  verify(mockReadModelRepo).updateUserName(UserId(1), "New Name")
}

test("event handler is idempotent") {
  val event = UserProfileUpdated(UserId(1), "New Name")

  // Process same event twice
  eventHandler.handle(event).unsafeRunSync()
  eventHandler.handle(event).unsafeRunSync()

  // Should only update once
  verify(mockReadModelRepo, times(1)).updateUserName(any, any)
}
```

#### Testing Pyramid

DIMDAh naturally produces a healthy testing pyramid:

```
        /\
       /  \  E2E (few, slow, through Interface layer)
      /____\
     /      \  Integration (moderate, services & subdomains)
    /________\
   /          \  Unit (many, fast, kernels & pure logic)
  /____________\
```

- **Many kernel unit tests**: Fast, comprehensive coverage of business logic
- **Moderate subdomain/service tests**: Verify coordination
- **Few interface/E2E tests**: Verify external contracts work

#### Best Practices

1. **Test behavior, not implementation**: Focus on what the component does, not how
2. **Use test data builders**: Create reusable fixtures for complex domain models
3. **Leverage property-based testing**: For kernels, use libraries like ScalaCheck
4. **Mock at architectural boundaries**: Mock repositories, not kernels
5. **Use architectural tests**: Tools like ArchUnit verify layer dependencies

### 2. Event-Driven Communication

**Problem**: DIMDAh's strict isolation prevents direct cross-context dependencies. But many operations require coordination — for example, when a user updates their profile, you may need to:
- Update CQRS read models in a separate context
- Notify other services of the change
- Trigger downstream workflows

**Solution**: **Domain events** serve as the communication mechanism between isolated contexts.

#### Event Emission

When a domain model is mutated, the responsible kernel or subdomain **emits a domain event** rather than directly calling other contexts:

```scala
// Anti-pattern: Direct call violates isolation
def updateUserProfile(userId: UserId, data: ProfileData): IO[Error, Unit] =
  for {
    _ <- userRepository.update(userId, data)
    _ <- cqrsService.updateReadModel(userId, data) // WRONG: breaks isolation
  } yield ()

// Correct: Emit event, let other contexts react
def updateUserProfile(userId: UserId, data: ProfileData): IO[Error, Unit] =
  for {
    _ <- userRepository.update(userId, data)
    _ <- eventPublisher.emit(UserProfileUpdated(userId, data))
  } yield ()
```

Domain events are **facts** — immutable records that something happened. They represent state changes, not commands.

#### Event Consumption

Other contexts subscribe to relevant events and react accordingly:

```scala
// CQRS read model context listens for user events
eventBus.subscribe[UserProfileUpdated] { event =>
  cqrsRepository.updateUserReadModel(event.userId, event.data)
}
```

Consumers are **independent** — the emitting context has no knowledge of who is listening or what they do with the event.

#### Implementation Options

**Internal Event Bus** (for monoliths or tightly-coupled deployments):
- Akka Event Stream
- In-memory pub/sub
- Fast, synchronous or async within process boundaries

**External Message Queue** (for distributed systems):
- Kafka, RabbitMQ, AWS SNS/SQS
- Durable, supports service boundaries
- Enables true microservices independence

**Key principle**: The choice between internal and external event mechanisms is a deployment decision. The domain logic remains unchanged.

#### CQRS Integration

CQRS (Command-Query Responsibility Segregation) fits naturally into DIMDAh:

- **Write side (Command)**: Core domain logic mutates models, emits domain events
- **Read side (Query)**: A separate context/service subscribes to events and builds optimized read models (denormalized, indexed for queries)
- **No direct dependency**: Write side doesn't know about read models; read side reacts to events

This preserves isolation while enabling specialized query performance.

#### Trade-offs and Considerations

- **Eventual consistency**: Subscribers process events asynchronously. The system must tolerate temporary inconsistency.
- **Event ordering**: If order matters, use techniques like event sequencing, partition keys, or version vectors.
- **Idempotency**: Event handlers must be idempotent — processing the same event twice should be safe.
- **Failure handling**: What happens if a subscriber fails? Use dead-letter queues, retries, or compensating actions.
- **Event versioning**: As the system evolves, event schemas change. Plan for backward/forward compatibility.

### 3. Transaction Boundaries and Consistency

Transaction management in DIMDAh follows the principle: **transactions respect isolation boundaries**. The scope of a transaction determines what consistency guarantees you can provide.

#### Within a Kernel (No Transactions Needed)

Kernels are purely functional — they don't require transactions. All state changes are explicit in return values.

```scala
// Pure function, no transaction needed
def calculateOrderTotal(items: List[OrderItem], tax: TaxRate): Money =
  items.map(_.price).sum * (1 + tax.value)
```

#### Within a Subdomain (Single Database Transaction)

When a subdomain operation involves multiple repository calls that must succeed or fail together, use a **single database transaction**:

```scala
def transferFunds(from: AccountId, to: AccountId, amount: Money): IO[TransferError, Unit] =
  transactionally {
    for {
      _ <- accountRepo.withdraw(from, amount)
      _ <- accountRepo.deposit(to, amount)
      _ <- auditRepo.logTransfer(from, to, amount)
    } yield ()
  }
```

**Characteristics**:
- ACID guarantees within the transaction
- All changes commit together or roll back together
- Transaction boundary is explicit in code
- Keep transactions short to avoid lock contention

#### Across Subdomains Within a Service (Two-Phase Commit or Event-Driven)

When coordinating multiple subdomains within the same service, you have two options:

**Option 1: Database Transaction (if sharing same database)**

If subdomains share a database (even with separate schemas), you can span a transaction:

```scala
def processOrder(orderId: OrderId): IO[OrderError, Unit] =
  transactionally {
    for {
      order <- orderSubdomain.markAsPaid(orderId)
      _ <- inventorySubdomain.reserveItems(order.items)
      _ <- shippingSubdomain.scheduleShipment(orderId)
    } yield ()
  }
```

**Option 2: Event-Driven Choreography**

If subdomains have separate databases, use events and eventual consistency:

```scala
def processPayment(orderId: OrderId): IO[PaymentError, Unit] =
  for {
    _ <- paymentRepo.recordPayment(orderId)
    _ <- eventPublisher.emit(PaymentCompleted(orderId))
  } yield ()

// Other subdomains react to the event
eventBus.subscribe[PaymentCompleted] { event =>
  inventorySubdomain.reserveItems(event.orderId)
}
```

#### Across Services (Distributed Transactions - Saga Pattern)

Services have **independent databases** and cannot share transactions. Use the **Saga pattern** for distributed coordination:

**Saga Orchestration Example**:

```scala
// Saga coordinator
def placeOrderSaga(order: Order): IO[SagaError, OrderId] =
  for {
    // Step 1: Reserve inventory
    _ <- inventoryService.reserve(order.items)
          .onError(e => IO.pure(())) // Nothing to compensate yet

    // Step 2: Process payment
    _ <- paymentService.charge(order.customerId, order.total)
          .onError(e =>
            inventoryService.releaseReservation(order.items) // Compensate step 1
          )

    // Step 3: Create order
    orderId <- orderService.create(order)
          .onError(e =>
            for {
              _ <- paymentService.refund(order.customerId, order.total)
              _ <- inventoryService.releaseReservation(order.items)
            } yield ()
          )
  } yield orderId
```

**Characteristics**:
- Each service operation is a local transaction
- Failures trigger compensating transactions
- System eventually reaches consistent state
- No ACID guarantees across services (eventual consistency)

**Saga Choreography Example** (event-driven):

```scala
// Order service
def createOrder(data: OrderData): IO[OrderError, OrderId] =
  for {
    orderId <- orderRepo.save(Order.pending(data))
    _ <- eventPublisher.emit(OrderCreated(orderId, data))
  } yield orderId

// Inventory service reacts
eventBus.subscribe[OrderCreated] { event =>
  inventoryRepo.reserve(event.items)
    .flatMap(_ => eventPublisher.emit(InventoryReserved(event.orderId)))
    .onError(_ => eventPublisher.emit(InventoryReservationFailed(event.orderId)))
}

// Order service reacts to failure
eventBus.subscribe[InventoryReservationFailed] { event =>
  orderRepo.markAsFailed(event.orderId)
}
```

#### Transactional Outbox Pattern

To ensure events are published reliably along with database changes, use the **transactional outbox pattern**:

```scala
def updateUserProfile(userId: UserId, data: ProfileData): IO[UpdateError, Unit] =
  transactionally {
    for {
      _ <- userRepo.update(userId, data)
      // Write event to outbox table in same transaction
      _ <- outboxRepo.insert(OutboxEvent(
        eventType = "UserProfileUpdated",
        payload = UserProfileUpdated(userId, data)
      ))
    } yield ()
  }

// Separate process reads outbox and publishes events
outboxProcessor.pollAndPublish { events =>
  events.foreach { event =>
    eventPublisher.emit(event.payload)
    outboxRepo.markAsPublished(event.id)
  }
}
```

**Benefits**:
- Events are never lost (written in same transaction as data)
- At-least-once delivery guarantee
- Decouples event publishing from business logic

#### Decision Matrix

| Scope | Pattern | Consistency | Use When |
|-------|---------|-------------|----------|
| Single kernel | Pure functions | N/A | All pure logic |
| Single subdomain | Database transaction | ACID | Multiple repo calls must be atomic |
| Multiple subdomains (same DB) | Database transaction | ACID | Shared database, strong consistency needed |
| Multiple subdomains (separate DBs) | Events + eventual consistency | Eventual | Independent databases, can tolerate eventual consistency |
| Multiple services | Saga pattern | Eventual | Distributed system, need coordinated workflow |
| Service + reliable events | Transactional outbox | Eventual + reliable | Must guarantee event delivery |

#### Best Practices

1. **Keep transactions short**: Long transactions cause lock contention
2. **Transaction boundaries should be explicit**: Use clear transaction demarcation in code
3. **Design for idempotency**: All operations should be safely retryable
4. **Compensating actions must be well-tested**: Saga failures can leave partial state
5. **Monitor saga progress**: Distributed sagas need observability (correlation IDs, tracing)
6. **Prefer choreography for simple flows**: Orchestration for complex multi-step workflows

### 4. Eventual Consistency

Systems that span multiple subdomains or services must tolerate eventual consistency. Each event represents a domain fact, and the system should be designed to recover gracefully from asynchronous propagation. Design contexts to handle stale or missing data gracefully, and provide mechanisms for eventual reconciliation.

### 5. Shared Model Strategy

**Core Principle**: Type duplication is acceptable; logic duplication is forbidden.

DIMDAh takes a pragmatic approach to sharing types across contexts while preventing tight coupling.

#### Internal vs Exposed Models

Each layer/context maintains **private internal models** for its own use and **exposes only specific views** to upper layers or other contexts:

- **Internal models**: Used for persistence, complex internal state, implementation details — these are **never exposed**
- **Exposed views**: Public data types tailored to each API or use case — these form the contract with consumers

**Example**: A User context might have:
- `UserEntity` (internal) — Full persistence model with database-specific fields, audit columns, etc. (**package-private**)
- `UserProfile` (exposed) — Public view for profile APIs
- `UserSummary` (exposed) — Lightweight view for list/search operations
- `UserSecurityContext` (exposed) — View containing only identity and permissions

Each exposed view is tailored to its use case. Internal models remain hidden, preventing coupling to implementation details.

#### Common Module for Shared Primitives

A **Common module** (or layer) contains universally shared primitive types that have no behavioral differences across contexts:

**Appropriate for Common**:
- Value objects: `UserId`, `Email`, `Money`, `Timestamp`
- Simple data structures: `Point(x, y)`, `Coordinate`, `Range`
- Primitive domain types that are universally the same

```scala
// Common module
case class UserId(value: Long) extends AnyVal
case class Email(value: String) extends AnyVal
case class Point(x: Double, y: Double)
```

**Not appropriate for Common**:
- Rich domain models with behavior (these belong in their owning context)
- Models that might have different interpretations in different contexts
- Anything with complex validation or business rules

#### When to Duplicate Types

**Duplicate types freely when**:
- Different contexts need different representations of the same concept
- Coupling would force one context to accommodate another's needs
- The type may evolve differently in each context
- The types serve different purposes even if they look similar now

**Example**: An `Order` in the Order Management context vs. an `Order` in the Shipping context might look similar but have different fields, validation rules, and purposes. Duplicate them.

#### When to Share Types

**Share types only when**:
- They are truly universal primitives (like `UserId`, `Money`)
- They have no behavior or business logic
- Their definition is stable and unlikely to diverge
- All contexts genuinely mean the exact same thing

#### Translation Between Contexts

When contexts communicate (via APIs or events), translate between internal and external representations explicitly:

```scala
// User context
object UserProfile {
  def fromEntity(entity: UserEntity): UserProfile =
    UserProfile(entity.id, entity.displayName, entity.email)
}

// Order context receives user data from User context
case class OrderUserInfo(userId: UserId, name: String)

// Translate from User context's view to Order context's view
def toOrderUserInfo(profile: UserProfile): OrderUserInfo =
  OrderUserInfo(profile.userId, profile.displayName)
```

Explicit translation makes dependencies visible and controllable.

#### Versioning and Evolution

- **Common primitives**: Version carefully, as changes ripple everywhere
- **Exposed views**: Version as needed for each API/use case (e.g., `UserProfileV2`)
- **Internal models**: Evolve freely without breaking consumers

When a shared primitive must change, consider:
- Adding a new version (`UserIdV2`) alongside the old one
- Using opaque types or phantom types to prevent accidental mixing
- Coordinating migration across dependent contexts

**Remember**: The goal is isolation and independent evolution. When in doubt, duplicate the type.

### 6. Documentation Discipline

Every subdomain must include a clear description of its purpose, invariants, and dependencies. Documentation is as essential as code — not only for developers but also for AI agents that collaborate on codebases.

### 7. Cross-Cutting Concerns

Cross-cutting concerns (logging, metrics, security, tracing) must be pervasive without polluting business logic. DIMDAh addresses this through a combination of structural patterns and effect system integration.

#### Using the Infrastructure Layer

As defined in the Layer Overview, the **Infrastructure layer** (Layer 0) provides the foundational abstractions for cross-cutting concerns. All other layers depend on these interfaces to access logging, metrics, security context, and tracing utilities.

By injecting these dependencies (via constructor injection, Reader monad, or ZIO environment), business logic remains:
- **Testable**: Mock implementations can be provided in tests
- **Pure**: No direct I/O or side effects in domain code
- **Portable**: Implementation details (log format, metric backend) can change without affecting business logic

#### Effect System Integration

Leverage the `IO[E, A]` effect type to carry cross-cutting context:

- **Correlation IDs**: Thread through operations for distributed tracing
- **User identity**: Security context flows with the effect
- **Trace spans**: Begin/end spans as part of effect composition
- **Request metadata**: Timestamps, source, environment information

This approach makes context propagation explicit and type-safe, without requiring global mutable state.

#### Decorator Pattern at Boundaries

Apply cross-cutting behavior at layer boundaries through composition:

- **Interface Layer**: HTTP middleware adds authentication, request logging, correlation ID generation
- **Service Layer**: Service wrappers add transaction boundaries, operation metrics, audit logging
- **Repository Layer**: Repository decorators add query logging, performance metrics, caching

**Benefit**: Business logic remains clean and focused. Cross-cutting concerns are added at composition/wiring time, not embedded in domain code.

#### Where Convention Is Still Required

Some concerns require discipline and code review:

- **No direct logging to stdout/stderr** in business logic — always use Logger abstraction
- **Sensitive data handling** — ensure PII is not logged or exposed inappropriately
- **Consistent metric naming** — establish naming conventions for observability
- **Error context enrichment** — ensure errors include sufficient debugging information

**Recommendation**: Use architectural tests (ArchUnit, custom linters) to enforce these conventions automatically where possible.

### 8. Performance Considerations

While DIMDAh prioritizes correctness and maintainability, performance remains important. Here's how to balance architectural purity with performance needs:

#### Areas of Potential Overhead

**Event Publishing**:
- **Overhead**: Event emission and handling adds latency
- **Mitigation**: Use in-memory event buses for same-process communication; only use external queues when crossing service boundaries
- **When to optimize**: If event handling is on the critical path, consider batching or async processing

**Type Translation**:
- **Overhead**: Translating between internal models and exposed views adds CPU cycles
- **Mitigation**: Keep translations simple (field mapping, no complex computation); use zero-cost abstractions where possible (value classes, inline functions)
- **When to optimize**: Profile first; translation overhead is usually negligible compared to I/O

**Effect Type Wrapping**:
- **Overhead**: Wrapping operations in `IO[E, A]` adds abstraction layers
- **Mitigation**: Modern effect systems (ZIO, Cats Effect) are highly optimized; the abstraction cost is minimal
- **When to optimize**: Only bypass effects for truly performance-critical hot paths (and document why)

**Transaction Boundaries**:
- **Overhead**: More granular transactions may mean more database round-trips
- **Mitigation**: Batch operations where possible; use connection pooling; consider read replicas for queries
- **When to optimize**: If transaction overhead is measurable, consider denormalization or caching

#### Performance-Friendly Patterns

**1. CQRS for Read Performance**

Separate read models from write models to optimize queries independently:

```scala
// Write side: normalized, transactional
def updateUserProfile(userId: UserId, data: ProfileData): IO[Error, Unit]

// Read side: denormalized, optimized for queries
def getUserDashboard(userId: UserId): IO[Error, DashboardView]
// Reads from pre-computed, indexed read model
```

**2. Caching at Boundaries**

Add caching at repository or service boundaries without polluting business logic:

```scala
// Repository with caching decorator
class CachedUserRepository(underlying: UserRepository, cache: Cache)
    extends UserRepository {
  def findById(id: UserId): IO[Nothing, Option[User]] =
    cache.get(id).flatMap {
      case Some(user) => IO.succeed(Some(user))
      case None => underlying.findById(id).tap(u => cache.put(id, u))
    }
}
```

**3. Async Event Handling**

Process non-critical events asynchronously to keep the main path fast:

```scala
def processOrder(order: Order): IO[OrderError, OrderId] =
  for {
    orderId <- orderRepo.save(order)
    // Critical path ends here
    _ <- eventPublisher.emit(OrderCreated(orderId)).fork // Async, non-blocking
  } yield orderId
```

**4. Batching and Bulk Operations**

Repository interfaces can support batch operations while maintaining abstraction:

```scala
trait UserRepository {
  def findById(id: UserId): IO[Nothing, Option[User]]
  def findByIds(ids: List[UserId]): IO[Nothing, Map[UserId, User]] // Batch query
}
```

#### When to Break Architectural Rules

In rare cases, performance requirements may justify breaking isolation:

**1. Hot Path Optimization**
- If profiling shows a critical bottleneck, consider optimizing that specific path
- Document the deviation clearly
- Keep the optimized path isolated

**2. Denormalization Across Contexts**
- If cross-context queries are too slow, consider controlled denormalization
- Maintain eventual consistency through events
- Document the trade-off

**3. Direct Database Joins**
- In read-heavy scenarios, complex joins across subdomain tables might be necessary
- Isolate these in read-model repositories
- Never use for write operations

#### Monitoring and Profiling

DIMDAh's boundaries make performance monitoring easier:

- **Instrument at layer boundaries**: Measure latency at Interface, Service, Subdomain, Repository
- **Track event handling time**: Monitor event processing latency and queue depth
- **Profile kernel logic**: Pure functions are easy to benchmark in isolation
- **Database query analysis**: Repository boundaries make it easy to log/analyze all queries

#### General Advice

1. **Measure first**: Don't optimize based on assumptions
2. **Optimize at boundaries**: Keep business logic pure; add caching, batching, etc. at architectural boundaries
3. **Document trade-offs**: If you deviate from the architecture for performance, document why
4. **Revisit regularly**: Performance requirements change; what was critical may become less so

**Remember**: Premature optimization is the enemy of maintainability. DIMDAh's clean boundaries make targeted optimization easier when you actually need it.

### 9. Runtime Orchestration

Runtime topology (whether modules are deployed as microservices or as parts of a monolith) is left open to project constraints. DIMDAh's logical service boundaries support both deployment models — you can start with a monolith and extract services later without major refactoring, as long as you respect the architectural boundaries.

---

## Summary

The **Domain-Isolated Modular Driver Architecture (DIMDAh)** provides a balance between flexibility and structure. By defining strong, semantic boundaries between kernels, contexts, and orchestrations, it allows teams — human and machine alike — to build and evolve large-scale systems without losing coherence or sanity.

### Core Tenets

1. **Cognitive load is the bottleneck** — Good architecture minimizes how much you need to understand to make changes safely
2. **Compiler over convention** — Encode rules in types and visibility modifiers, not just documentation
3. **Domain errors are data** — Make failure modes explicit in function signatures
4. **Type duplication is acceptable; logic duplication is forbidden** — Duplicate types to prevent coupling, never duplicate business logic
5. **Events enable isolation** — Cross-context communication happens through immutable events, not direct calls
6. **Transactions respect boundaries** — Transaction scope aligns with isolation boundaries

### Architectural Layers

- **Layer 0: Infrastructure** — Cross-cutting abstractions (logging, metrics, etc.)
- **Layer 1: Core (Domain)** — Domain models, kernels, repository interfaces, DAOs
- **Layer 2: Subdomain** — Bounded contexts combining kernels into cohesive units
- **Layer 3: Service** — Orchestration layer, represents deployment boundaries
- **Layer 4: Interface** — External APIs and protocol handlers

### When to Use DIMDAh

DIMDAh is well-suited for:

- **Complex business domains** with substantial business logic
- **Long-lived systems** that will evolve over years
- **Small to medium teams** that need to move quickly without breaking things
- **AI-assisted development** where clear boundaries help both humans and machines reason about code
- **Systems with clear subdomain boundaries** where isolation provides value

DIMDAh may be overkill for:

- Simple CRUD applications with minimal business logic
- Prototypes and experiments with short lifespans
- Systems where performance constraints require tight coupling

### Next Steps

When adopting DIMDAh:

1. **Identify your subdomains** — Map your business capabilities to bounded contexts
2. **Start with kernels** — Implement pure business logic in isolated, testable units
3. **Define repository interfaces** — Abstract persistence behind clear contracts
4. **Use package-private visibility** — Let the compiler enforce boundaries
5. **Introduce events gradually** — Start with synchronous calls, evolve to events as you scale
6. **Document invariants** — Make subdomain boundaries and rules explicit

DIMDAh is a living architecture — adapt it to your context while preserving the core principle: **minimize cognitive load through enforced isolation**.

---

## Naming Conventions

Consistent naming makes the architecture self-documenting. These conventions help developers and AI agents quickly understand the role of each component.

### Layer 0: Infrastructure

**Interfaces/Traits**:
- Pattern: `<Capability>` (e.g., `Logger`, `Metrics`, `Clock`)
- Should describe what they do, not how they do it
- Use present tense, active voice

**Examples**:
```scala
trait Logger[F[_]]
trait Metrics[F[_]]
trait TimeProvider[F[_]]
trait IdGenerator[F[_]]
```

### Layer 1: Core (Domain)

**Domain Models**:
- Pattern: Singular noun representing the entity
- Use domain language, not technical terms
- Examples: `User`, `Order`, `Product`, `Invoice`

**Value Objects**:
- Pattern: `<Domain><Concept>` or just `<Concept>`
- Examples: `UserId`, `Email`, `Money`, `OrderStatus`

**Kernels**:
- Pattern: `<Domain>Kernel` or `<Domain>Logic`
- Contains pure business logic
- Examples: `UserKernel`, `OrderKernel`, `PricingLogic`

**Repository Interfaces**:
- Pattern: `<Entity>Repository`
- Examples: `UserRepository`, `OrderRepository`, `ProductRepository`

**DAOs (package-private)**:
- Pattern: `<Entity>DAO` or `<Entity>DataAccess`
- Examples: `UserDAO`, `OrderDAO`

**Domain Errors**:
- Pattern: `<Context><ErrorCondition>`
- Use past tense or noun form
- Examples: `UserNotFound`, `InsufficientBalance`, `InvalidEmailFormat`, `OrderAlreadyShipped`

**Domain Events**:
- Pattern: `<Entity><ActionInPastTense>`
- Always past tense (facts that happened)
- Examples: `UserRegistered`, `OrderPlaced`, `PaymentCompleted`, `InventoryReserved`

### Layer 2: Subdomain

**Context/Subdomain Modules**:
- Pattern: `<Domain>Context` or `<Domain>Subdomain`
- Examples: `UserContext`, `OrderSubdomain`, `InventoryContext`

**Subdomain Services**:
- Pattern: `<Domain>Service` or `<Domain>Manager`
- Examples: `UserService`, `OrderManager`

**Context Events**:
- Pattern: `<Context><Event>` (same as domain events)
- Examples: `UserProfileUpdated`, `OrderStatusChanged`

### Layer 3: Service

**Service Modules**:
- Pattern: `<BusinessCapability>Service`
- Should reflect business capabilities, not technical groupings
- Examples: `CheckoutService`, `FulfillmentService`, `CustomerManagementService`

**Process/Workflow Objects**:
- Pattern: `<Action>Process` or `<Action>Workflow`
- Examples: `OrderPlacementProcess`, `RefundWorkflow`, `UserOnboardingProcess`

**Service Errors**:
- Pattern: `<Service><Error>`
- Examples: `CheckoutFailed`, `FulfillmentError`, `InvalidCheckoutRequest`

### Layer 4: Interface

**Controllers/Endpoints**:
- Pattern: `<Entity>Controller` or `<Entity>Endpoints`
- Examples: `UserController`, `OrderEndpoints`, `ProductApi`

**DTOs (Data Transfer Objects)**:
- Pattern: `<Entity><Purpose>DTO` or `<Entity><Purpose>Request/Response`
- Examples: `UserRegistrationRequest`, `OrderSummaryResponse`, `CreateProductDTO`

**View Models**:
- Pattern: `<Entity><View>View` or `<Entity><Purpose>ViewModel`
- Examples: `UserProfileView`, `OrderListView`, `DashboardViewModel`

### Common Module

**Shared Value Objects**:
- Pattern: Simple, descriptive names
- Examples: `UserId`, `Money`, `Email`, `Timestamp`, `Point`, `Coordinate`

### File/Package Organization

**Scala Example**:
```
com.example.project/
├── infrastructure/
│   ├── Logger.scala
│   ├── Metrics.scala
│   └── TimeProvider.scala
├── domain/                               // or "core" - Domain Layer
│   ├── user/
│   │   ├── User.scala                    // Domain model
│   │   ├── UserId.scala                  // Value object
│   │   ├── UserKernel.scala              // Pure logic
│   │   ├── UserRepository.scala          // Interface (public)
│   │   ├── UserDAO.scala                 // Implementation (package-private)
│   │   ├── UserRegistered.scala          // Domain event
│   │   └── UserErrors.scala              // Error types
│   └── order/
│       ├── Order.scala
│       ├── OrderKernel.scala
│       └── ...
├── subdomain/
│   ├── user/
│   │   └── UserContext.scala
│   └── order/
│       └── OrderSubdomain.scala
├── service/
│   ├── checkout/
│   │   ├── CheckoutService.scala
│   │   └── OrderPlacementProcess.scala
│   └── fulfillment/
│       └── FulfillmentService.scala
└── interface/
    ├── http/
    │   ├── UserController.scala
    │   ├── OrderController.scala
    │   └── dto/
    │       ├── UserRegistrationRequest.scala
    │       └── OrderSummaryResponse.scala
    └── grpc/
        └── ...
```

### Function Naming

**Kernel Functions** (pure logic):
- Pattern: Verb describing the operation
- Examples: `validateEmail`, `calculateTotal`, `applyDiscount`, `checkEligibility`

**Repository Functions**:
- Pattern: CRUD-style operations
- Examples: `findById`, `findByEmail`, `save`, `update`, `delete`, `exists`
- Query methods: `findBy<Criteria>`, `listBy<Criteria>`

**Service/Subdomain Functions**:
- Pattern: Business operation verbs
- Examples: `registerUser`, `placeOrder`, `processPayment`, `shipOrder`, `cancelSubscription`

**Event Handlers**:
- Pattern: `on<Event>` or `handle<Event>`
- Examples: `onUserRegistered`, `handleOrderPlaced`, `onPaymentCompleted`

### General Guidelines

1. **Be explicit over clever**: `UserRegistrationRequest` is better than `RegReq`
2. **Use domain language**: If the business calls it "fulfillment", don't call it "shipping"
3. **Avoid technical prefixes**: Don't prefix interfaces with `I` (use Scala/Kotlin/Java conventions)
4. **Package visibility matters**: If it's `package-private`, the name can be more implementation-focused
5. **Consistency within context**: Once you choose a convention (e.g., `Context` vs `Subdomain`), stick with it

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        External World                           │
│              (HTTP Clients, gRPC, Message Queues)               │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Layer 4: Interface                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │ Controllers  │  │     DTOs     │  │   Protocol   │           │
│  │  Endpoints   │  │ View Models  │  │   Handlers   │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
└───────────────────────────┬─────────────────────────────────────┘
                            │ Calls
                            ▼
┌────────────────────────────────────────────────────────────────┐
│                    Layer 3: Service                            │
│                  (Orchestration Layer)                         │
│                                                                │
│  ┌────────────────────┐        ┌────────────────────┐          │
│  │  Service A         │        │  Service B         │          │
│  │  ┌──────────────┐  │        │  ┌──────────────┐  │          │
│  │  │   Process    │  │        │  │   Process    │  │          │
│  │  │   Workflow   │  │        │  │   Workflow   │  │          │
│  │  └──────────────┘  │        │  └──────────────┘  │          │
│  └────────────────────┘        └────────────────────┘          │
│           │                              │                     │
│           │ Coordinates                  │                     │
│           ▼                              ▼                     │
│   ┌─────────────────────────────────────────────────┐          │
│   │      Inter-Service Communication via Events     │          │
│   └─────────────────────────────────────────────────┘          │
└───────────────────────────┬────────────────────────────────────┘
                            │ Uses
                            ▼
┌────────────────────────────────────────────────────────────────┐
│                    Layer 2: Subdomain                          │
│                  (Bounded Contexts)                            │
│                                                                │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐    │
│  │  Subdomain A   │  │  Subdomain B   │  │  Subdomain C   │    │
│  │  ┌──────────┐  │  │  ┌──────────┐  │  │  ┌──────────┐  │    │
│  │  │ Context  │  │  │  │ Context  │  │  │  │ Context  │  │    │
│  │  │  Units   │  │  │  │  Units   │  │  │  │  Units   │  │    │
│  │  └──────────┘  │  │  └──────────┘  │  │  └──────────┘  │    │
│  │  Combines      │  │                │  │                │    │
│  │  kernels       │  │                │  │                │    │
│  └────────────────┘  └────────────────┘  └────────────────┘    │
│           │                   │                   │            │
│           │  Event Publisher  │                   │            │
│           └───────────┬───────┴───────────────────┘            │
└───────────────────────┼────────────────────────────────────────┘
                        │ Uses
                        ▼
┌────────────────────────────────────────────────────────────────┐
│                Layer 1: Core (Domain)                          │
│                  (Domain Foundation)                           │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Domain Models (User, Order, Product, ...)              │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │   │
│  │  │   Kernel A   │  │   Kernel B   │  │   Kernel C   │   │   │
│  │  │ (Pure Logic) │  │ (Pure Logic) │  │ (Pure Logic) │   │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Repository Interfaces (public)                         │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │   │
│  │  │UserRepo     │  │OrderRepo    │  │ProductRepo  │      │   │
│  │  │(interface)  │  │(interface)  │  │(interface)  │      │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  DAOs (package-private implementations)                 │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │   │
│  │  │  UserDAO    │  │  OrderDAO   │  │ ProductDAO  │      │   │
│  │  │  (hidden)   │  │  (hidden)   │  │  (hidden)   │      │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                             │                                  │
│                             │ Persists to                      │
│                             ▼                                  │
│                    ┌─────────────────┐                         │
│                    │    Database     │                         │
│                    └─────────────────┘                         │
└────────────────────────────────────────────────────────────────┘
                            │ Depends on
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Layer 0: Infrastructure                      │
│                  (Cross-Cutting Abstractions)                   │
│                                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐         │
│  │  Logger  │  │ Metrics  │  │ Security │  │  Tracing │         │
│  │  (trait) │  │ (trait)  │  │ Context  │  │  (trait) │         │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘         │
│                                                                 │
│           All layers can depend on these abstractions           │
└─────────────────────────────────────────────────────────────────┘

Event Flow (Cross-Context Communication):
═════════════════════════════════════════

Subdomain A                    Subdomain B
    │                               │
    │ 1. Domain operation           │
    │    (e.g., updateUser)         │
    ▼                               │
┌─────────┐                         │
│ Mutate  │                         │
│  State  │                         │
└────┬────┘                         │
     │                              │
     │ 2. Emit event                │
     │    (UserUpdated)             │
     ▼                              │
┌──────────────┐                    │
│ EventPublisher│───────────────────┼──────────┐
└──────────────┘                    │          │
                                    │          │
                       ┌────────────▼─────┐    │
                       │   Event Bus      │    │
                       │  (Internal or    │    │
                       │   External)      │    │
                       └────────┬─────────┘    │
                                │              │
                                │ 3. Subscribe │
                                ▼              │
                           ┌────────────┐      │
                           │  Handler   │◄─────┘
                           └─────┬──────┘
                                 │
                                 │ 4. React
                                 ▼
                            Update CQRS
                            Trigger workflow
                            Notify other contexts
```

### Key Architectural Flows

**1. Request Processing (Top-Down)**:
```
User Request
    → Interface Layer (validates, translates to domain)
    → Service Layer (orchestrates workflow)
    → Subdomain Layer (coordinates kernels)
    → Core (Domain) Layer (executes business logic, persists via repositories)
    → Infrastructure (logging, metrics)
```

**2. Cross-Context Communication (Event-Driven)**:
```
Subdomain A changes state
    → Emits domain event
    → Event bus distributes
    → Subdomain B subscribes and reacts
    → No direct dependency between A and B
```

**3. Dependency Direction**:
```
Interface → Service → Subdomain → Core (Domain) → Infrastructure
                                        ↓
                                    Database
```

All layers can depend on Infrastructure (Layer 0), but never the reverse.

---

