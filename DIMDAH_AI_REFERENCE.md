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
- **Exception**: Port and Gate nodes are designed to be accessed from outside the context — Port (query) by any foreign Lobe or Orch; Gate (transaction) by foreign Orchs only

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
- **Can access**: Its own Kernels; other contexts' **Ports** (read/query only)
- **Cannot**: Access Repos directly; call another context's Gate; cause cross-context mutations (that's Orch's job)
- **Visibility**: Package-private to the context (Orch only)

### Orch + Port + Gate
- **Port**: Public **query** interface of the context. Exposes read-only operations. Accessible by any foreign Lobe or Orch.
- **Gate**: Public **transaction** interface of the context. Exposes business transaction operations. Accessible by foreign Orchs only — never by foreign Lobes.
- **Orch**: Implements both Port and Gate. Delegates to Lobes. Manages cross-context transactions (Sagas).
- **Can access**: Its own Lobes; other contexts' Ports and Gates
- **Cannot**: Access own Kernels or Repos directly
- **Visibility**: Port, Gate, and Orch are public

### Layer Collapsing

When there is **no Lobe-level logic** (no cross-kernel decisions, no external Port queries), Lobe may be omitted — Orch calls Kernel directly. This is intentional and communicates that the context has no higher-order business rules yet.

As soon as Lobe-level logic appears, reintroduce the Lobe, make Kernel package-private to it, and have Orch call Lobe instead of Kernel.

---

## Access Rules Summary Table

| Layer | Can Access | Cannot Access | Accessed By |
|-------|-----------|---------------|-------------|
| **Repo** | Database | Anything else | Owner Kernel only |
| **Kernel** | Own Repos | Other Kernels, Lobes, Ports | Owner Lobe (or Orch if collapsed) |
| **Lobe** | Own Kernels; external **Ports** (read) | Repos; external **Gates**; cross-context mutations | Context Orch |
| **Orch** | Own Lobes; external Ports and **Gates** | Own Kernels, Repos | Interface tier; other Orchs |

**Port** = query interface (foreign Lobes and Orchs may call)
**Gate** = transaction interface (foreign Orchs only — never foreign Lobes)

---

## Dependency Rules

```
Interface tier
    │
  Port (query — public) or Gate (transaction — public)
    │
  Orch ──► own Lobes ──► own Kernels ──► own Repos ──► DB
    │
  other context Ports (reads)
  other context Gates (cross-context mutations — Orch only)

All tiers ──► Infrastructure (abstractions only)
```

**NEVER**:
- Orch accessing Kernel or Repo directly (bypasses business rules)
- Lobe calling another context's Gate (only Orch may trigger cross-context transactions)
- Lobe mutating another context's state (use events or let Orch orchestrate)
- Lobe accessing Repo directly (bypasses Kernel invariants)
- Port or Gate leaking internal model types (Kernel/Repo-level)
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
  ├── Port.scala           ← public (query interface)
  ├── Gate.scala           ← public (transaction interface; foreign Lobes cannot access)
  ├── Orch.scala           ← public (implements Port and Gate)
  ├── lobes/
  │   ├── x/
  │   │   ├── XLobe.scala                  ← private[context]
  │   │   └── kernels/
  │   │       ├── XKernel.scala            ← private[x]
  │   │       └── XRepo.scala              ← private[x]
  │   └── y/
  │       ├── YLobe.scala                  ← private[context]
  │       └── kernels/
  │           ├── a/
  │           │   ├── AKernel.scala        ← private[y]
  │           │   └── repos/
  │           │       ├── ARepo1.scala     ← private[a]
  │           │       └── ARepo2.scala     ← private[a]
  │           └── b/
  │               ├── BKernel.scala        ← private[y]
  │               └── BRepo.scala          ← private[b]
  └── subcontexts/         ← optional; for units hidden from the outer world
      └── sub-a/
          ├── Port.scala   ← private[context] (parent + sibling Lobes/Orchs only)
          ├── Gate.scala   ← private[context] (parent + sibling Orchs only)
          ├── Orch.scala   ← private[context]
          └── lobes/
              └── ...
```

---

## Naming Conventions

| Component | Pattern | Examples |
|-----------|---------|---------|
| Port (query) | `<Context>Port` or `Port` (within package) | `UserPort`, `CheckoutPort` |
| Gate (transaction) | `<Context>Gate` or `Gate` (within package) | `UserGate`, `CheckoutGate` |
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

## Subcontexts

A **subcontext** is a context nested inside another context, hidden from the outside world. All SET rules apply equally. The distinction is access control:

- A subcontext is **not accessible by foreign contexts** — only its parent and sibling subcontexts.
- A subcontext has both Port (query) and Gate (transaction).
- The subcontext's **Port** is accessible by parent and sibling Orchs and Lobes.
- The subcontext's **Gate** is accessible only by parent and sibling Orchs.

**When to use**: When a bounded unit of domain logic must be hidden from the outer world and its access controlled entirely within the parent context.

In Scala, enforce via `private[parentContextPackage]` on the subcontext's Port, Gate, and Orch.

---

## Common Anti-Patterns

| Anti-pattern | Why it's wrong | Fix |
|---|---|---|
| Orch calling Kernel directly (when Lobe exists) | Bypasses Lobe business rules | Orch → Lobe → Kernel |
| Lobe accessing Repo | Bypasses Kernel invariants | Lobe → Kernel → Repo |
| Lobe calling another context's Gate | Cross-context mutations belong to Orch | Delegate to Orch or emit event |
| Lobe mutating another context via Port | Port is query-only | Emit event or delegate to Orch |
| Port or Gate exposing Kernel/Repo types | Leaks internals | Define Port/Gate-level types |
| Using exceptions for domain errors | Callers can't handle exhaustively | Use `IO[E, A]` with sealed ADT |
| Shared mutable state between Kernels | Creates hidden coupling | Each Kernel owns its Repos |
| Business logic in Interface tier | Protocol and domain coupled | Move logic to Lobe |
| Generic error types (`String`, `Throwable`) | Non-exhaustive handling | Use sealed ADT per domain |
| Foreign context accessing subcontext directly | Violates subcontext isolation | Route through parent context's Port or Gate |

---

## "Where Does This Code Belong?"

**Is it a raw DB query or persistence operation?**
→ **Repo**

**Is it an invariant check, DB-to-domain translation, or aggregate rule?**
→ **Kernel**

**Is it a business decision requiring data from outside the context, or coordinating multiple Kernels?**
→ **Lobe**

**Is it a read-only operation exposed to other contexts?**
→ **Port** (query interface)

**Is it a business transaction exposed to other contexts?**
→ **Gate** (transaction interface)

**Is it implementing the context's public API or managing a cross-context transaction?**
→ **Orch**

**Is it a cross-cutting concern (logging, metrics, tracing)?**
→ **Infrastructure** (interface/trait only)

**Is it translating domain results to HTTP/gRPC/etc., or handling auth?**
→ **Interface tier**

---

## Code Review Checklist

- [ ] Each layer accesses only what SET permits (see access rules table)
- [ ] Lobes call only foreign **Ports** (queries) — never foreign **Gates** (transactions)
- [ ] Cross-context mutations are managed by Orch, not Lobe
- [ ] Port error types are public sealed ADTs; Kernel/Lobe errors are package-private
- [ ] Inner layer errors are translated at each boundary — not propagated raw
- [ ] Events are past tense and immutable; handlers are idempotent
- [ ] DB-level model types do not appear outside the Repo
- [ ] Port/Gate-exposed models are distinct from internal models
- [ ] Subcontexts are not accessed by foreign contexts (scoped to parent package)
- [ ] No business logic in Interface tier
- [ ] Technical exceptions not used for domain outcomes
- [ ] Layer collapsing (Lobe omitted) is justified — no Lobe-level logic exists yet
- [ ] Naming follows conventions (Repo, Kernel, Lobe, Orch, Port, Gate suffixes)
