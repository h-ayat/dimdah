# DIMDAh Architecture Reference for AI Agents

**Purpose**: Quick reference for AI agents working in DIMDAh codebases. Focus on rules, patterns, and constraints.

**Full specifications**:
- Domain tier: [`dimdah.md`](./dimdah.md)
- Error handling: [`error-handling.md`](./error-handling.md)
- Infrastructure tier: [`infrastructure.md`](./infrastructure.md)
- Interface tier: [`interface.md`](./interface.md)

---

## System Tiers

| Tier | Responsibility |
|------|---------------|
| **Infrastructure** | Cross-cutting abstractions only (logging, metrics, security, tracing). No implementations, no business logic. |
| **Domain** | All business logic. Organized into contexts via SET. **This reference focuses here.** |
| **Interface** | External-facing APIs. Protocol translation only. No business logic. |

Dependency direction: `Interface → Domain → Infrastructure`

---

## Core Principles

1. **Minimize cognitive load** — Changes must be localized; understanding the whole system should never be required
2. **Compiler over convention** — Package-private visibility and typed errors enforce rules structurally
3. **Domain errors are data** — Encode expected failures in signatures: `IO[DomainError, Result]`
4. **Type duplication OK, logic duplication forbidden** — Duplicate types freely to prevent coupling; never duplicate business logic
5. **Events for cross-context side effects** — Emit events rather than calling other contexts directly for mutations
6. **Transactions respect boundaries** — Cross-context consistency uses Sagas, not shared transactions

---

## Strict Encapsulation Tree (SET)

The Domain tier is a **tree of packages**. The rule:

> **A component can only be accessed by its direct parent in the tree.**

- No sibling access (lobes cannot call each other)
- No skip-level access (Orch cannot reach Kernel directly, bypassing Lobe)
- **Exception**: Port nodes are designed to be accessed from outside the context — they are the public surface

Enforced via **package-private visibility modifiers** at each level.

---

## Domain Layers

Each context in the Domain tier has four layers:

### Repo
- **Does**: Communicates with storage (DB, cache). Holds DB-level models.
- **Can access**: Database only
- **Cannot**: Contain business logic; be accessed by anything other than its owner Kernel
- **Visibility**: Package-private to its Kernel's package

### Kernel
- **Does**: Maintains internal consistency of its aggregate. Translates DB models ↔ domain models. Enforces invariants.
- **Can access**: Its own Repos only
- **Cannot**: Talk to other Kernels, Lobes, Ports, or anything external
- **Visibility**: Package-private to its Lobe's package (or to Orch if Lobe is collapsed)

### Lobe
- **Does**: Decision-making and business rules. Coordinates Kernels. Queries external context Ports for data.
- **Can access**: Its own Kernels; other contexts' Ports (read/query only)
- **Cannot**: Access Repos directly; cause cross-context mutations (that's Orch's job)
- **Visibility**: Package-private to the context (Orch only)

### Orch + Port
- **Port**: Public interface of the context. Sealed contract. All external access goes through Port.
- **Orch**: Implements Port. Delegates to Lobes. Manages cross-context transactions (Sagas).
- **Can access**: Its own Lobes; other contexts' Ports
- **Cannot**: Access own Kernels or Repos directly
- **Visibility**: Port and Orch are public

### Layer Collapsing

When there is **no Lobe-level logic** (no cross-kernel decisions, no external Port queries), Lobe may be omitted — Orch calls Kernel directly. This is intentional and communicates that the context has no higher-order business rules yet.

As soon as Lobe-level logic appears, reintroduce the Lobe, make Kernel package-private to it, and have Orch call Lobe instead of Kernel.

---

## Access Rules Summary Table

| Layer | Can Access | Cannot Access | Accessed By |
|-------|-----------|---------------|-------------|
| **Repo** | Database | Anything else | Owner Kernel only |
| **Kernel** | Own Repos | Other Kernels, Lobes, Ports | Owner Lobe (or Orch if collapsed) |
| **Lobe** | Own Kernels; external Ports (read) | Repos; cross-context mutations | Context Orch |
| **Orch** | Own Lobes; external Ports | Own Kernels, Repos | Interface tier; other Orchs |

---

## Dependency Rules

```
Interface tier
    │
  Port (context boundary — public)
    │
  Orch ──► own Lobes ──► own Kernels ──► own Repos ──► DB
    │
  other context Ports (for reads or cross-context coordination)

All tiers ──► Infrastructure (abstractions only)
```

**NEVER**:
- Orch accessing Kernel or Repo directly (bypasses business rules)
- Lobe mutating another context's state (use events or let Orch orchestrate)
- Lobe accessing Repo directly (bypasses Kernel invariants)
- Port leaking internal model types (Kernel/Repo-level)
- Business logic in the Interface tier

---

## Error Handling Pattern

> Full guide: [`error-handling.md`](./error-handling.md)

All business logic returns `IO[E, A]`:
- `E` = domain error (sealed ADT, explicit in signature)
- `A` = success value
- Technical exceptions travel a separate channel

### Error vocabulary per layer

| Layer | Error type scope |
|-------|-----------------|
| Repo | Package-private (never exposed) |
| Kernel | Package-private to Lobe; translated from Repo errors |
| Lobe | Package-private to context; may wrap Kernel errors |
| Port | **Public** — the error vocabulary consumers depend on |

Always translate errors at boundaries. Inner error types must never leak through the Port.

### Pattern
```scala
// Error types are sealed ADTs
sealed trait UserError
case object EmailAlreadyTaken extends UserError
case object PlanNotAvailable  extends UserError

// Domain logic returns typed errors
def registerWithPlan(email: String, planId: PlanId): IO[UserError, UserId]

// Interface tier translates to protocol
case UserError.EmailAlreadyTaken => Response(409, "Email already in use")
```

---

## Event-Driven Communication

Events are used for **cross-context side effects** (e.g., updating a read model, triggering a downstream workflow). They are facts — past tense, immutable.

```scala
// ❌ Wrong: direct call from Lobe into another context's internals
def updateProfile(id: UserId, data: ProfileData): IO[Error, Unit] =
  for {
    _ <- kernel.applyUpdate(id, data)
    _ <- cqrsService.updateReadModel(id, data)  // breaks isolation
  } yield ()

// ✅ Correct: emit event; other contexts subscribe and react
def updateProfile(id: UserId, data: ProfileData): IO[Error, Unit] =
  for {
    _ <- kernel.applyUpdate(id, data)
    _ <- eventPublisher.emit(ProfileUpdated(id, data))
  } yield ()
```

**Event naming**: `<Entity><PastTenseVerb>` — `UserRegistered`, `OrderPlaced`, `PaymentCompleted`

**Event handlers** must be idempotent. Pattern: `on<Event>` or `handle<Event>`.

---

## Transaction Boundaries

| Scope | Pattern | Consistency |
|-------|---------|-------------|
| Within a Kernel | DB transaction over multiple Repo calls | ACID |
| Multiple Kernels in a Lobe (same DB) | DB transaction | ACID |
| Cross-context (separate DBs) | Events + eventual consistency | Eventual |
| Cross-context coordinated workflow | Saga + compensating actions (in Orch) | Eventual |
| Reliable event delivery | Transactional outbox | Eventual + reliable |

Cross-context transactions are **always managed by Orch**, never by Lobe.

---

## Package Structure

```
context/
  ├── Port.scala           ← public
  ├── Orch.scala           ← public (implements Port)
  └── lobes/
      ├── x/
      │   ├── XLobe.scala                  ← private[context]
      │   └── kernels/
      │       ├── XKernel.scala            ← private[x]
      │       └── XRepo.scala              ← private[x]
      └── y/
          ├── YLobe.scala                  ← private[context]
          └── kernels/
              ├── a/
              │   ├── AKernel.scala        ← private[y]
              │   └── repos/
              │       ├── ARepo1.scala     ← private[a]
              │       └── ARepo2.scala     ← private[a]
              └── b/
                  ├── BKernel.scala        ← private[y]
                  └── BRepo.scala          ← private[b]
```

---

## Naming Conventions

| Component | Pattern | Examples |
|-----------|---------|---------|
| Port | `<Context>Port` or `Port` (within package) | `UserPort`, `CheckoutPort` |
| Orch | `<Context>Orch` or `Orch` | `UserOrch`, `CheckoutOrch` |
| Lobe | `<Domain>Lobe` | `UserLobe`, `CheckoutLobe` |
| Kernel | `<Domain>Kernel` | `UserKernel`, `AccountKernel` |
| Repo | `<Domain>Repo` | `UserRepo`, `AccountRepo` |
| Domain models (Port-exposed) | Singular noun | `UserProfile`, `OrderSummary` |
| Value objects | `<Concept>` or `<Domain><Concept>` | `UserId`, `Email`, `Money` |
| Domain errors | `<Context><Condition>` | `EmailAlreadyTaken`, `PlanNotAvailable` |
| Domain events | `<Entity><PastTenseVerb>` | `UserRegistered`, `OrderPlaced` |
| Repo functions | CRUD style | `findById`, `insert`, `update`, `delete` |
| Kernel functions | Business operation verb | `register`, `deactivate`, `applyDiscount` |
| Lobe functions | Intent-revealing | `registerWithPlan`, `checkoutItems` |
| Port/Orch functions | Business capability | `getProfile`, `placeOrder` |
| Event handlers | `on<Event>` or `handle<Event>` | `onUserRegistered`, `handleOrderPlaced` |

---

## Shared Models Strategy

- **Internal models** (Repo/Kernel): package-private, never exposed outside the context
- **Port models**: public data types returned/accepted by Port methods — the external contract
- **Common module**: universal primitives only (`UserId`, `Email`, `Money`) — no behavior, no business rules

**Duplicate types freely** when contexts need different representations of the same concept. Coupling contexts through shared rich models is worse than duplication.

---

## Testing Strategy

| Layer | Test type | Dependencies |
|-------|-----------|-------------|
| **Repo** | Integration — real or in-memory DB | Database |
| **Kernel** | Unit — mock Repos | Mocked Repo |
| **Lobe** | Unit/integration — mock Kernels + mock Ports | Mocked Kernel, mocked external Ports |
| **Orch** | Orchestration — mock Lobes + mock external Ports | Mocked Lobe, verify compensation paths |

**Test pyramid**: Many Kernel tests → moderate Lobe tests → few Orch tests → minimal Interface tests.

**Event handlers**: Test each handler in isolation; always verify idempotency.

---

## Common Anti-Patterns

| Anti-pattern | Why it's wrong | Fix |
|---|---|---|
| Orch calling Kernel directly (when Lobe exists) | Bypasses Lobe business rules | Orch → Lobe → Kernel |
| Lobe accessing Repo | Bypasses Kernel invariants | Lobe → Kernel → Repo |
| Lobe mutating another context | Breaks cross-context isolation | Emit event or delegate to Orch |
| Port exposing Kernel/Repo types | Leaks internals | Define Port-level types |
| Using exceptions for domain errors | Callers can't handle exhaustively | Use `IO[E, A]` with sealed ADT |
| Shared mutable state between Kernels | Creates hidden coupling | Each Kernel owns its Repos |
| Business logic in Interface tier | Protocol and domain coupled | Move logic to Lobe |
| Generic error types (`String`, `Throwable`) | Non-exhaustive handling | Use sealed ADT per domain |

---

## "Where Does This Code Belong?"

**Is it a raw DB query or persistence operation?**
→ **Repo**

**Is it an invariant check, DB-to-domain translation, or aggregate rule?**
→ **Kernel**

**Is it a business decision requiring data from outside the context, or coordinating multiple Kernels?**
→ **Lobe**

**Is it implementing the context's public API or managing a cross-context transaction?**
→ **Orch**

**Is it a cross-cutting concern (logging, metrics, tracing)?**
→ **Infrastructure** (interface/trait only)

**Is it translating domain results to HTTP/gRPC/etc., or handling auth?**
→ **Interface tier**

---

## Code Review Checklist

- [ ] Each layer accesses only what SET permits (see access rules table)
- [ ] Port error types are public sealed ADTs; Kernel/Lobe errors are package-private
- [ ] Inner layer errors are translated at each boundary — not propagated raw
- [ ] Orch manages all cross-context transactions; Lobe does not mutate other contexts
- [ ] Events are past tense and immutable; handlers are idempotent
- [ ] DB-level model types do not appear outside the Repo
- [ ] Port-exposed models are distinct from internal models
- [ ] No business logic in Interface tier
- [ ] Technical exceptions not used for domain outcomes
- [ ] Layer collapsing (Lobe omitted) is justified — no Lobe-level logic exists yet
- [ ] Naming follows conventions (Repo, Kernel, Lobe, Orch, Port suffixes)
