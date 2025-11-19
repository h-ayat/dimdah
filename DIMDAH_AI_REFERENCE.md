# DIMDAh Architecture Reference for AI Agents

**Purpose**: Quick reference for AI agents working in DIMDAh codebases. Focus on rules, patterns, and constraints.

---

## Core Principles

1. **Minimize cognitive load** — Changes should be localized; understanding the entire codebase should not be required
2. **Compiler over convention** — Use types, visibility modifiers, and effect systems to enforce rules
3. **Domain errors are data** — Encode expected failures in function signatures: `IO[DomainError, Result]`
4. **Type duplication OK, logic duplication forbidden** — Duplicate types freely to prevent coupling; never duplicate business logic
5. **Events for cross-context communication** — Never call other contexts directly; emit events instead
6. **Transactions respect boundaries** — Transaction scope aligns with isolation boundaries

---

## Architecture Layers

### Layer 0: Infrastructure (Support)
- **Purpose**: Cross-cutting abstractions (logging, metrics, security, tracing)
- **Contains**: Interfaces/traits only, no implementations
- **Rule**: All layers can depend on this; it depends on nothing
- **Examples**: `Logger[F[_]]`, `Metrics[F[_]]`, `TimeProvider[F[_]]`

### Layer 1: Core (Domain)
- **Purpose**: Domain models, business logic, persistence abstractions
- **Contains**:
  - Domain Models (immutable, validated)
  - Kernels (pure business logic, no I/O)
  - Repository interfaces (public)
  - DAOs (package-private implementations)
  - Event Publishers
- **Rule**:
  - Kernels must be pure (no side effects)
  - DAOs are package-private; only repository interfaces are public
  - No dependencies on higher layers
- **Examples**: `User`, `UserKernel`, `UserRepository`, `UserDAO` (hidden)

### Layer 2: Subdomain (Bounded Contexts)
- **Purpose**: Combine kernels into cohesive business capabilities
- **Contains**:
  - Context units (combine multiple kernels)
  - Subdomain events
- **Rule**:
  - Can use multiple kernels from Core
  - Expose narrow, intention-revealing APIs
  - Hide internal implementation details
  - No direct dependencies on other subdomains
- **Examples**: `UserContext`, `OrderSubdomain`

### Layer 3: Service (Orchestration)
- **Purpose**: Coordinate subdomains into workflows; deployment boundaries
- **Contains**:
  - Process/workflow objects
  - Service-level orchestration
  - Error composition
- **Rule**:
  - Each service = potential microservice (independently deployable)
  - No business logic (only orchestration)
  - Communicate with other services via events or APIs
  - Multiple subdomains can live within one service
- **Examples**: `CheckoutService`, `FulfillmentService`

### Layer 4: Interface (External APIs)
- **Purpose**: Expose system to external world
- **Contains**:
  - Controllers/Endpoints
  - DTOs, View Models
  - Protocol handlers (HTTP, gRPC, etc.)
- **Rule**:
  - Translate between external protocols and domain
  - Delegate all work to Service or Subdomain layers
  - Never contain business logic
- **Examples**: `UserController`, `OrderEndpoints`

---

## Dependency Rules

```
Interface → Service → Subdomain → Core → Infrastructure
                                    ↓
                                Database

All layers → Infrastructure
```

**NEVER**:
- Higher layers depending on lower layer implementations (only interfaces)
- Direct cross-subdomain calls (use events)
- Business logic in Interface or Service layers
- Public access to DAOs (they must be package-private)

---

## Error Handling Pattern

### IO[E, A] Effect Type

All business logic functions return: `IO[E, A]` where:
- `E` = Domain error type (explicit, in signature)
- `A` = Success type
- Technical exceptions wrapped separately from domain errors

### Example
```scala
// Domain errors are explicit in signature
def registerUser(data: UserData): IO[UserRegistrationError, UserId]

// Error types are ADTs
sealed trait UserRegistrationError
case object EmailAlreadyExists extends UserRegistrationError
case object InvalidEmailFormat extends UserRegistrationError
case class ValidationFailed(field: String) extends UserRegistrationError
```

### Rules
- Domain errors are **expected outcomes**, not exceptions
- Always encode domain errors in the type signature
- Technical failures (DB connection, network) are handled separately by the effect system
- Callers must handle all domain error cases

---

## Event-Driven Communication

### When to Use Events
- Cross-subdomain coordination
- Cross-service communication
- Triggering side effects without coupling (CQRS, notifications, etc.)

### Pattern
```scala
// ❌ WRONG: Direct call to another context
def updateUserProfile(userId: UserId, data: ProfileData): IO[Error, Unit] =
  for {
    _ <- userRepo.update(userId, data)
    _ <- cqrsService.updateReadModel(userId, data)  // Violates isolation!
  } yield ()

// ✅ CORRECT: Emit event, let other contexts react
def updateUserProfile(userId: UserId, data: ProfileData): IO[Error, Unit] =
  for {
    _ <- userRepo.update(userId, data)
    _ <- eventPublisher.emit(UserProfileUpdated(userId, data))
  } yield ()

// Other context subscribes
eventBus.subscribe[UserProfileUpdated] { event =>
  cqrsRepository.updateUserReadModel(event.userId, event.data)
}
```

### Event Naming
- Pattern: `<Entity><ActionInPastTense>`
- Always past tense (facts that happened)
- Examples: `UserRegistered`, `OrderPlaced`, `PaymentCompleted`

### Event Handlers
- Must be idempotent (safe to process same event multiple times)
- Pattern: `on<Event>` or `handle<Event>`
- Examples: `onUserRegistered`, `handleOrderPlaced`

---

## Transaction Boundaries

| Scope | Pattern | Use When |
|-------|---------|----------|
| Single kernel | Pure functions (no transaction) | All pure logic |
| Single subdomain | Database transaction | Multiple repo calls must be atomic |
| Multiple subdomains (same DB) | Database transaction | Shared database, strong consistency needed |
| Multiple subdomains (separate DBs) | Events + eventual consistency | Independent databases |
| Multiple services | Saga pattern | Distributed coordination needed |
| Service + reliable events | Transactional outbox | Must guarantee event delivery |

### Transactional Outbox (Reliable Events)
```scala
// Write event to outbox in same transaction as data change
transactionally {
  for {
    _ <- userRepo.update(userId, data)
    _ <- outboxRepo.insert(OutboxEvent("UserProfileUpdated", payload))
  } yield ()
}

// Separate process publishes from outbox
outboxProcessor.pollAndPublish { events =>
  events.foreach { event =>
    eventPublisher.emit(event.payload)
    outboxRepo.markAsPublished(event.id)
  }
}
```

---

## Naming Conventions

### Domain Models
- `User`, `Order`, `Product` (singular nouns, domain language)

### Value Objects
- `UserId`, `Email`, `Money`, `OrderStatus`

### Kernels
- `UserKernel`, `OrderKernel`, `PricingLogic`

### Repositories
- Interface: `UserRepository` (public)
- DAO: `UserDAO` (package-private)

### Errors
- `UserNotFound`, `InsufficientBalance`, `InvalidEmailFormat`

### Events
- `UserRegistered`, `OrderPlaced`, `PaymentCompleted` (past tense)

### Subdomains
- `UserContext`, `OrderSubdomain`, `InventoryContext`

### Services
- `CheckoutService`, `FulfillmentService` (business capabilities)

### Controllers
- `UserController`, `OrderEndpoints`

### DTOs
- `UserRegistrationRequest`, `OrderSummaryResponse`

### Functions
- Kernels: `validateEmail`, `calculateTotal` (verbs)
- Repositories: `findById`, `save`, `update` (CRUD)
- Services: `registerUser`, `placeOrder` (business operations)
- Event handlers: `onUserRegistered`, `handleOrderPlaced`

---

## Shared Models Strategy

### Internal vs Exposed Models
- **Internal models**: Package-private, used for persistence (e.g., `UserEntity`)
- **Exposed views**: Public, tailored to each use case (e.g., `UserProfile`, `UserSummary`)
- **Rule**: Never expose internal models; always create views

### Common Module
Contains universal primitives:
- `UserId`, `Email`, `Money`, `Timestamp`, `Point`
- **Use for**: Simple value objects with no behavior
- **Don't use for**: Rich domain models with business rules

### When to Duplicate Types
- Different contexts need different representations
- Coupling would force one context to accommodate another
- Types may evolve differently
- **Rule**: Duplicate types freely; when in doubt, duplicate

### When to Share Types
- Truly universal primitives
- No behavior or business logic
- Definition is stable

---

## Visibility Modifiers (Enforcement)

### Public
- Repository interfaces
- Exposed domain models/views
- Subdomain APIs
- Infrastructure abstractions

### Package-Private
- DAOs (concrete persistence implementations)
- Internal domain model details
- Helper utilities
- Internal constructors/validators

**Rule**: Use package-private to enforce abstraction boundaries at compile time.

---

## Testing Strategy

### Kernel Tests
- Pure unit tests, no mocking
- Test business logic in isolation
- Fast (microseconds)

### Repository Tests
- Integration tests with in-memory DB or testcontainers
- Test actual persistence behavior

### Subdomain Tests
- Mock repositories
- Test kernel coordination
- Verify event emission

### Service Tests
- End-to-end workflow tests
- Mix of real and mocked dependencies

### Interface Tests
- HTTP/gRPC contract tests
- Mock service layer

### Event Handler Tests
- Test each handler in isolation
- Verify idempotency
- Mock dependencies

---

## Common Anti-Patterns to Avoid

❌ **Business logic in Interface or Service layers**
- Business rules belong in Kernels

❌ **Direct cross-context calls**
- Use events instead

❌ **Public DAOs**
- Keep them package-private

❌ **Bypassing repository interfaces**
- Always use abstractions

❌ **Mutable domain models**
- Keep them immutable

❌ **Exceptions for domain errors**
- Use `IO[DomainError, A]` pattern

❌ **Global state or singletons for domain logic**
- Use dependency injection

❌ **Leaking infrastructure concerns into domain**
- Keep kernels pure, no logging/metrics in business logic

---

## Package Structure Example

```
com.example.project/
├── infrastructure/          (Layer 0)
│   ├── Logger.scala
│   ├── Metrics.scala
│   └── TimeProvider.scala
├── core/                    (Layer 1)
│   ├── user/
│   │   ├── models.scala                // Domain model, errors, etc... (public)
│   │   ├── UserKernel.scala          // Pure logic (public)
│   │   ├── UserRepo.scala      // Interface (public)
│   │   ├── UserDao.scala             // DAO (package-private)
│   │   ├── events.scala      // Event (public)
│   └── order/
│       └── ...
├── subdomain/               (Layer 2)
│   ├── user/
│   │   └── UserContext.scala
│   └── order/
│       └── OrderSubdomain.scala
├── service/                 (Layer 3)
│   ├── checkout/
│   │   └── CheckoutService.scala
│   └── fulfillment/
│       └── FulfillmentService.scala
└── interface/               (Layer 4)
    └── http/
        ├── UserController.scala
        └── dto/
            └── UserRegistrationRequest.scala
```

---

## Quick Decision Tree

### "Where does this code belong?"

**Is it pure business logic?**
→ Yes: Put it in a Kernel (Layer 1)

**Does it coordinate multiple kernels?**
→ Yes: Put it in a Subdomain (Layer 2)

**Does it orchestrate multiple subdomains?**
→ Yes: Put it in a Service (Layer 3)

**Does it handle HTTP/external protocols?**
→ Yes: Put it in Interface (Layer 4)

**Is it a cross-cutting concern?**
→ Yes: Put an abstraction in Infrastructure (Layer 0)

### "Should this be a new subdomain or part of an existing one?"

**Are they tightly coupled in the business domain?**
→ Yes: Same subdomain

**Can they have separate databases?**
→ Yes: Separate subdomains

**Would changes in one require changes in the other?**
→ Often: Same subdomain

### "Should this be a new service?"

**Could it be deployed independently as a microservice?**
→ Yes: Separate service

**Does it have its own database?**
→ Yes: Separate service

**Is it loosely coupled to other services?**
→ Yes: Separate service

---

## Performance Optimization Patterns

### CQRS
- Write side: Normalized, transactional (Core)
- Read side: Denormalized, optimized for queries (separate context reacting to events)

### Caching
- Add at repository boundaries using decorator pattern
- Never in kernels

### Async Event Processing
```scala
eventPublisher.emit(OrderCreated(orderId)).fork // Non-blocking
```

### Batching
```scala
trait UserRepository {
  def findById(id: UserId): IO[Nothing, Option[User]] 
  def findByIds(ids: List[UserId]): IO[Nothing, Map[UserId, User]]
}
```

**Rule**: Keep business logic pure; add optimizations at architectural boundaries.

---

## Key Reminders

1. **Favor isolation over DRY** — Duplicate types to prevent coupling
2. **Make illegal states unrepresentable** — Use types to enforce invariants
3. **Events are facts, not commands** — Past tense, immutable
4. **Transactions stay within boundaries** — Don't span services
5. **Package-private is your friend** — Use it to enforce encapsulation
6. **Kernels are pure** — No I/O, no side effects, just logic
7. **When in doubt, add a layer of indirection** — Better isolated than coupled

---

## Checklist for Code Reviews

- [ ] Business logic is in Kernels (pure functions)
- [ ] Domain errors are in function signatures
- [ ] DAOs are package-private
- [ ] No direct cross-context calls (events used instead)
- [ ] No business logic in Interface or Service layers
- [ ] Events are past tense, immutable
- [ ] Repository interfaces used (not concrete DAOs)
- [ ] Proper layer dependencies (no upward dependencies)
- [ ] Transaction boundaries respected
- [ ] Event handlers are idempotent
- [ ] Naming conventions followed
- [ ] Internal models not exposed (views used instead)

---

**For questions or clarifications, refer to the full specification: `dimdah.md`**
