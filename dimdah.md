# DIMDAh — Domain-Isolated Modular Driven Architecture

**Version: 0.2 · 2026-09-24**

Normative specification. Every rule in DIMDAh lives here and nowhere else —
this is the file projects vendor and agents load. [`rationale.md`](./rationale.md)
argues *why* and shows worked examples; it states no rules, and where the two
appear to disagree, this file wins.

Reference language is **Scala 3**: the visibility rules assume it. Other
languages apply the same rules with their own enforcement mechanism.

`IO[E, A]` below names the effect type — a result, a typed domain error `E`, or
a technical exception on a separate channel. A project may bind it to its own
type (`ZIO[Any, E, A]`, `EitherT[IO, E, A]`, `Result<A, E>`, a custom `X[E, A]`)
and record that once; the rules are unchanged.

---

## 1. Tiers

| Tier | Responsibility |
|------|---------------|
| **Interface** | External-facing APIs. Protocol translation only. No decisions. |
| **Domain** | All business logic, organized into contexts under SET. |
| **Infrastructure** | Cross-cutting abstractions only. No implementations, no business logic. |

Dependency direction: `Interface → Domain → Infrastructure`. Every tier may
depend on Infrastructure *abstractions*; nothing depends on Interface.

---

## 2. Core principles

1. **Cognitive load is the bottleneck** — changing one part must never require
   understanding the whole.
2. **Compiler over convention** — package-private visibility and typed errors
   enforce the rules structurally.
3. **Domain errors are data** — expected failures are in the signature:
   `IO[DomainError, Result]`.
4. **Duplicate types freely; never duplicate a decision** — two copies of a
   type cost nothing; two copies of a rule diverge.
5. **Events for cross-context side effects** — emit facts, don't reach in.
6. **Transactions respect boundaries** — cross-context consistency uses Sagas,
   never shared transactions.

---

## 3. Domain tier

### 3.1 Strict Encapsulation Tree (SET)

The Domain tier is a tree of packages. The rule:

> **A component can only be accessed by its direct parent in the tree.**

- No sibling access — two Lobes never call each other.
- No skip-level access — Orch never reaches Kernel.
- **One exception**: Port and Gate are designed for access across context
  boundaries — Port (query) by any foreign Lobe or Orch; Gate (transaction) by
  foreign Orchs only.

Enforced with package-private visibility at each level.

### 3.2 Context

A *context* is a bounded unit of domain responsibility. Each context has four
layers — Repo, Kernel, Lobe, Orch — plus a **Port** (query interface) and a
**Gate** (transaction interface) forming its public surface. A context with no
mutations has no Gate.

### 3.3 The four layers

#### Repo
- **Does**: talks to storage. Defines DB-level models.
- **Can access**: the database only.
- **Cannot**: contain business logic.
- **Accessed by**: its owner Kernel only.
- **Visibility**: package-private to its Kernel's package.
- DB-level models never leave the Repo's package.

#### Kernel
- **Does**: maintains the invariants of one aggregate. Translates DB models ↔
  domain models.
- **Can access**: its own Repos only.
- **Cannot**: talk to other Kernels, Lobes, Ports, or anything external.
- **Accessed by**: its owner Lobe only.
- **Visibility**: package-private to its Lobe's package.
- A Kernel must be understandable in complete isolation.

#### Lobe
- **Does**: decision-making and business rules. Coordinates its Kernels.
  Queries foreign context **Ports** for data it needs to decide.
- **Can access**: its own Kernels; foreign **Ports** (read only).
- **Cannot**: access Repos; call a foreign **Gate**; mutate another context.
- **Accessed by**: its context's Orch only.
- **Visibility**: package-private to the context.

#### Orch, Port, Gate
- **Port**: public **query** interface. Read-only operations. Callable by any
  foreign Lobe or Orch. Named by the *nature of the operation* (read-only),
  never by which layer happens to call it — a query stays a Port even when only
  an Orch calls it.
- **Gate**: public **transaction** interface. Mutations. Callable by foreign
  Orchs and the Interface tier only — never by a foreign Lobe.
- **Orch**: implements Port and Gate. Delegates to Lobes. Manages
  cross-context/cross-service transactions (Sagas) by sequencing calls and
  triggering compensations — never by evaluating business rules itself.
- **Can access**: its own Lobes; foreign Ports and Gates — **to route or
  sequence, never to decide**.
- **Cannot**: access its own Kernels or Repos; contain validation, permission
  checks, or derived-value computation.
- **Visibility**: Port, Gate and Orch are public.
- **Files**: Port and Gate each in their own file (`Port.scala`, `Gate.scala`)
  — never together, never inside `Orch.scala`.

### 3.4 Orch vs Lobe: what counts as orchestration

Being *structurally allowed* to call a foreign Port/Gate from Orch does not
make the logic *around* that call Orch's. SET says which edges are legal; it
says nothing about what a layer may compute once it holds the data. Orch's only
legitimate jobs:

1. Sequencing calls — its own Lobe, foreign Ports/Gates, external services.
2. Reacting to success/failure with the next call, including compensations.
3. Locking and idempotency around the cross-context transaction.

Anything that inspects data and produces a decision — a permission check, an
amount validation, "does this account have enough credit", building a derived
business value — is business logic and belongs to **Lobe**, even if Lobe must
query that same foreign Port itself to get the data.

**Litmus test**: strip every `if`/`ensure`/validation/derived-value computation
out of the Orch method. What remains — locking, calling Lobe, calling a
Gate/Port/external service, branching on success vs. failure to compensate — is
legitimately Orch's. If nothing remains, the whole method belongs in Lobe and
Orch becomes a single delegating call.

```scala
// ❌ Orch fetches foreign data and evaluates business rules itself
def debit(proId: ProId, extensionId: ExtensionId, amount: Rial): IO[DebitError, Unit] =
  for {
    extension <- extensionPort.getExtensionById(extensionId).someOrWrapDie()
    ()        <- IO.ensure(extension.requiredPerms.contains(Permission.ProWallet), DebitError.InvalidAmount)
    pricing   <- extensionPort.getExtensionLastPrice(extension)
    ()        <- lobe.ensureAmount(pricing, amount, DebitError.InvalidAmount)
    ()        <- doDebit(...)
  } yield ()

// ✅ Lobe owns the decision (querying the foreign Port itself);
//    Orch only locks, delegates, and sequences the cross-service call
def debit(proId: ProId, extensionId: ExtensionId, amount: Rial): IO[DebitError, Unit] =
  referenceLock.lockAndFail(lockObject, DebitError.InvalidReference) { _ =>
    for {
      prepared <- lobe.prepareDebit(proId, extensionId, amount)
      ()       <- doDebit(prepared.transactionId, prepared.debitType, ...)
    } yield ()
  }
```

This holds whether the foreign dependency is another in-repo context or a
genuinely external service: Orch may call it, only Lobe may reason about what
it returns.

### 3.5 No layer collapsing

Every context always has an Orch and at least one Lobe. No layer is ever
omitted. A layer with nothing of its own to do still exists and forwards.

```scala
// ✅ Pass-through layers are fine
// Orch
def getProfile(id: UserId): IO[ProfileError, UserProfile] = lobe.getProfile(id)
// Lobe
def getProfile(id: UserId): IO[ProfileError, UserProfile] = kernel.getProfile(id)
```

Orch never calls Kernel directly, and Kernel is always package-private to its
Lobe.

### 3.6 Access rules

| Layer | Can access | Cannot access | Accessed by |
|-------|-----------|---------------|-------------|
| **Repo** | Database | Anything else | Owner Kernel only |
| **Kernel** | Own Repos | Other Kernels, Lobes, Ports | Owner Lobe only |
| **Lobe** | Own Kernels; foreign **Ports** (read) | Repos; foreign **Gates**; cross-context mutations | Context Orch |
| **Orch** | Own Lobes; foreign Ports and **Gates** | Own Kernels, Repos | Interface tier; other Orchs |

```
Interface tier
    │
  Port (query — public) or Gate (transaction — public)
    │
  Orch ──► own Lobes ──► own Kernels ──► own Repos ──► DB
    │
  foreign Ports (reads)
  foreign Gates (cross-context mutations — Orch only)

All tiers ──► Infrastructure (abstractions only)
```

**Never**:
- Orch accessing Kernel or Repo directly.
- Orch deciding anything (see 3.4).
- Omitting Orch or Lobe because it would only forward.
- Lobe calling a foreign Gate, or mutating another context through a Port.
- Lobe accessing a Repo.
- Port or Gate exposing Kernel/Repo-level types.
- A decision in the Interface tier (see 4.6).

### 3.7 Subcontexts

A **subcontext** is a context nested inside another and hidden from the rest of
the system. All SET rules apply unchanged. Only access differs:

- Foreign contexts cannot reach it — only its parent and sibling subcontexts.
- Its **Port** is accessible by the parent Orch, the parent's Lobes, and
  sibling subcontexts' Orchs and Lobes.
- Its **Gate** is accessible by the parent Orch and sibling subcontexts' Orchs.

Use it when a bounded unit of logic must be hidden from the outer world and its
exposure governed by the parent. It is not a default decomposition strategy. In
Scala, scope its Port, Gate and Orch `private[parentContext]`.

### 3.8 Package structure

```
context/
  ├── Port.scala           ← public (query interface; own file)
  ├── Gate.scala           ← public (transaction interface; own file)
  ├── Orch.scala           ← public (implements Port and Gate; always present)
  ├── lobes/               ← always present, even if every Lobe only forwards
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
  └── subcontexts/         ← optional; units hidden from the outer world
      └── sub-a/
          ├── Port.scala   ← private[context]
          ├── Gate.scala   ← private[context]
          ├── Orch.scala   ← private[context]
          └── lobes/
```

### 3.9 Models

- **Internal models** (Repo/Kernel): package-private, never exposed.
- **Port/Gate models**: public types in the context's contract.
- **Common module**: universal primitives only (`UserId`, `Email`, `Money`) —
  no behavior, no business rules.

Duplicate types freely when contexts need different representations of the same
concept. Coupling contexts through one shared rich model is worse than
duplication.

### 3.10 Events

Cross-context side effects travel as **domain events** — facts, past tense,
immutable. The emitter has no knowledge of subscribers.

- Naming: `<Entity><PastTenseVerb>` — `UserRegistered`, `OrderPlaced`.
- Handlers: `on<Event>` / `handle<Event>`, and always idempotent.
- Reliable delivery uses a transactional outbox.

### 3.11 Transactions

| Scope | Pattern | Consistency |
|-------|---------|-------------|
| Within a Kernel | DB transaction over its Repo calls | ACID |
| Multiple Kernels in a Lobe (same DB) | DB transaction | ACID |
| Cross-context (separate DBs) | Events | Eventual |
| Cross-context coordinated workflow | Saga + compensations, in Orch | Eventual |
| Reliable event delivery | Transactional outbox | Eventual + reliable |

Cross-context transactions are always managed by Orch, never by Lobe.

---

## 4. Interface tier

The Interface tier exposes the Domain tier to the outside world. It translates;
it never decides.

### 4.1 Sub-layers

| Sub-layer | Owns |
|-----------|------|
| **Transport** | Protocol mechanics: routing, (de)serialization, headers/cookies, status emission, framework wiring, **authentication** |
| **Contract** | What client and server agree on — see 4.2 |
| **Impl** | Server-side implementation of the contract: translation between contract types and Port/Gate calls |

Call direction: `Transport → Impl → Port/Gate`. Transport knows the contract and
the framework; Impl knows the contract and the domain; the contract knows
neither framework nor domain.

### 4.2 The contract has two halves

**Structure** — per endpoint: name, input types, result type, error type, and
the session/actor type. One trait per endpoint group; an endpoint is a
function. Framework-free and protocol-free.

**Transport binding** — how a call crosses processes for one protocol: codecs,
the status code per error type, path, verb, cookie mechanics. It is part of the
contract, beneath Structure, and lives beside it in the contract module.

Rules:
- The Structure half depends on nothing but the language and shared primitives.
- The binding half may depend on a serialization library and a protocol
  vocabulary (status codes, verbs) — **never** on a client or server framework.
  A second binding (gRPC, WebSocket) is then additive.
- Endpoint groups are organized by **audience, expressed as packages**
  (`api.admin.UserProfile`, `api.user.UserProfile`) — never as name prefixes.
  The package is also a visibility boundary the compiler can enforce.
- Two audiences may declare near-identical endpoints and types. That is
  duplication of *shape*, which is free. They must call the same Gate/Port
  operation, because the *rule* may exist only once.

### 4.3 Impl

An Impl method may only: map contract types to domain types, call one or more
Port/Gate operations, map domain errors to contract errors, and map the result
back. Sequencing across contexts is Orch's job, not Impl's.

### 4.4 Errors are data

- Each contract error is its own type — a tagged object or case class — and the
  error channel of an endpoint is the set of errors it can return.
- The binding maps each error type to its protocol response (status code, body).
  A wire-level tag identifies the error type in the body, and that tag is
  written by hand, never derived from a class name — a rename must not break a
  client.
- The technical-exception channel gets one generic response (500) carrying no
  detail.

### 4.5 Authentication vs authorization

- **Authentication** — establishing *who is calling* — happens in **Transport**
  (session cookie, token, header) and nowhere else.
- **Authorization** — deciding *whether that caller may do this* — is a domain
  decision and belongs in the Domain tier, never in the Interface tier. When it
  is complex enough, it earns its own context (see `DIMDAH-OPEN-1`).
- The acting identity is an **explicit parameter** of the domain operation —
  never ambient (no thread-local, no implicit request scope). A permission rule
  that isn't visible in the signature isn't reviewable.

### 4.6 The Interface-tier prohibition, precisely

"No business logic in the Interface tier" is imprecise. The checkable rule:

> **An Interface-tier component contains no decision that two audiences could
> answer differently.**

Mechanical translation that every audience performs identically — session →
actor, DTO → domain type, domain error → status code — is translation and
belongs here. Anything an admin API and a user API could answer differently is
a decision and belongs in a Lobe.

### 4.7 Composition root

The runnable application sits in the Interface tier and holds the composition
root: the single place that reads configuration and constructs and injects
concrete implementations. No other code reads the environment.

---

## 5. Infrastructure tier

- Holds **abstractions only** — logging, metrics, tracing, clock, id
  generation, security context, storage drivers' interfaces.
- Contains no business logic and no protocol knowledge.
- Concretions are constructed only at the composition root (4.7) and injected;
  Domain and Interface code depends on the interfaces alone.
- Configuration is a typed value supplied by the composition root — never read
  ambiently by Domain or Interface code.

---

## 6. Error handling

All business logic returns `IO[E, A]`:

- `E` — a domain error: an expected business outcome, in the signature.
- `A` — the success value.
- Technical exceptions — unexpected infrastructure failures — travel the
  effect's separate channel and are never used for business outcomes.

### 6.1 Error types

- Domain errors are sealed ADTs (or unions of distinct types where the language
  supports exhaustive handling). Never `String`, never `Throwable`, never a
  generic error type.
- Naming: `<Context><Condition>` — `EmailAlreadyTaken`, `InsufficientBalance`.

### 6.2 Vocabulary per layer

| Layer | Error scope |
|-------|-------------|
| Repo | Package-private; never exposed |
| Kernel | Package-private to its Lobe; translated from Repo errors |
| Lobe | Package-private to the context; may wrap Kernel errors |
| Port / Gate | **Public** — the vocabulary consumers depend on |

Translate at every boundary. An inner layer's error type never leaks through
the Port or Gate, and a context's error type never leaks through the Interface
tier's contract unchanged — the contract has its own error vocabulary (4.4).

### 6.3 Domain error vs technical exception

| | Domain error | Technical exception |
|--|-------------|---------------------|
| Nature | Expected business outcome | Unexpected infrastructure failure |
| In the signature | Yes | No |
| Caller must handle | Yes, at compile time | No — global handler |
| Response | Business-meaningful status | Generic 500 |

Domain code never maps an error to a protocol response; that translation exists
only in the Interface tier.

---

## 7. Naming

| Component | Pattern | Examples |
|-----------|---------|----------|
| Port | `<Context>Port` or `Port` in-package | `UserPort` |
| Gate | `<Context>Gate` or `Gate` in-package | `UserGate` |
| Orch | `<Context>Orch` or `Orch` | `UserOrch` |
| Lobe | `<Domain>Lobe` | `CheckoutLobe` |
| Kernel | `<Domain>Kernel` | `AccountKernel` |
| Repo | `<Domain>Repo` | `AccountRepo` |
| Port/Gate models | Singular noun | `UserProfile`, `OrderSummary` |
| Value objects | `<Concept>` | `UserId`, `Email`, `Money` |
| Domain errors | `<Context><Condition>` | `EmailAlreadyTaken` |
| Domain events | `<Entity><PastTenseVerb>` | `OrderPlaced` |
| Contract trait | `<Group>Api` | `UserProfileApi` |
| Contract binding | `<Group><Protocol>` | `UserProfileRest` |
| Contract impl | `<Group>ApiImpl` | `UserProfileApiImpl` |
| Transport handler | `<Group>Routes` | `UserProfileRoutes` |
| Repo functions | CRUD style | `findById`, `insert` |
| Kernel functions | Business operation verb | `register`, `deactivate` |
| Lobe functions | Intent-revealing | `registerWithPlan` |
| Port/Gate functions | Business capability | `getProfile`, `placeOrder` |
| Event handlers | `on<Event>` / `handle<Event>` | `onUserRegistered` |

Be explicit over clever; use the domain's language; no `I` prefixes on
interfaces; keep one convention per context.

---

## 8. Testing

| Layer | Test type | Dependencies |
|-------|-----------|--------------|
| **Repo** | Integration — real or in-memory DB | Database |
| **Kernel** | Unit — mocked Repos | Mocked Repo |
| **Lobe** | Unit/integration — mocked Kernels and foreign Ports | Mocked Kernel, mocked Ports |
| **Orch** | Orchestration — mocked Lobes and foreign Ports; verify compensations | Mocked Lobe |
| **Impl** | Unit — mocked Port/Gate; no protocol involved | Mocked Gate/Port |
| **Transport** | Few, end-to-end through the real protocol | The wired application |

Test pyramid: many Kernel tests → moderate Lobe tests → few Orch tests →
minimal Interface tests. Mock at architectural boundaries, never internal
helpers. Event handlers are tested in isolation, always including idempotency.

---

## 9. Anti-patterns

| Anti-pattern | Why it's wrong | Fix |
|---|---|---|
| Orch calling Kernel directly | Bypasses Lobe business rules | Orch → Lobe → Kernel, even if Lobe only forwards |
| Omitting Orch or Lobe because it would only forward | Collapsed layers blur SET boundaries and must be reintroduced later | Keep the layer as a pass-through |
| Orch evaluating rules on data from a foreign Port/Gate | The edge is legal, the decision is not; it escapes Lobe's test suite | Move fetch + decision into Lobe; Orch sequences only |
| Port and Gate in one file, or inside `Orch.scala` | Blurs the query/transaction split | `Port.scala` and `Gate.scala`, one each |
| Lobe accessing a Repo | Bypasses Kernel invariants | Lobe → Kernel → Repo |
| Lobe calling a foreign Gate | Cross-context mutations belong to Orch | Delegate to Orch or emit an event |
| Lobe mutating through a foreign Port | Port is query-only | Emit an event or delegate to Orch |
| Port or Gate exposing Kernel/Repo types | Leaks internals | Define Port/Gate-level types |
| Exceptions used for domain errors | Callers cannot handle exhaustively | `IO[E, A]` with a sealed ADT |
| Generic error types (`String`, `Throwable`) | No exhaustive handling | One ADT per domain vocabulary |
| Shared mutable state between Kernels | Hidden coupling | Each Kernel owns its Repos |
| A decision in the Interface tier | Two audiences will answer it differently one day | Move it to a Lobe (4.6) |
| Authorization checked in Transport or Impl | Same rule, re-derived per endpoint | Domain decides; pass the actor explicitly |
| Ambient caller identity (thread-local, implicit scope) | Permission rules invisible in signatures | Explicit actor parameter |
| Contract structure depending on a framework | Contract stops being shareable with clients | Framework stays in Transport |
| Wire tags derived from class names | A rename breaks clients silently | Hand-written tag constants |
| Foreign context reaching a subcontext | Breaks subcontext isolation | Route through the parent's Port or Gate |

---

## 10. Where does this code belong?

- Raw persistence or a DB query → **Repo**
- An invariant, a DB↔domain translation, an aggregate rule → **Kernel**
- A decision, coordination of Kernels, or a rule needing foreign data → **Lobe**
- A read-only operation exposed to other contexts → **Port**
- A business transaction exposed to other contexts → **Gate**
- Implementing the context's public surface, or a cross-context saga → **Orch**
  (sequencing, locking, compensation only)
- The client/server agreement — endpoint shape, errors, codecs, status → **Contract**
- Mapping contract types to Port/Gate calls → **Impl**
- Routing, serialization, cookies, authentication → **Transport**
- Reading configuration, constructing concretions → **composition root**
- A cross-cutting abstraction (logging, metrics, tracing, clock) → **Infrastructure**

---

## 11. Review checklist

- [ ] Each layer accesses only what 3.6 permits
- [ ] Every context has an Orch and Lobe(s), even if they only forward
- [ ] Orch methods contain no validation, permission check, or derived value —
      only sequencing, locking, compensation (3.4)
- [ ] Lobes call foreign **Ports** only — never foreign **Gates**
- [ ] Cross-context mutations are managed by Orch
- [ ] Port and Gate are separate files; naming reflects query vs. mutation
- [ ] Port/Gate error types are public; Kernel/Lobe error types are package-private
- [ ] Errors are translated at every boundary, not propagated raw
- [ ] DB-level models do not appear outside the Repo
- [ ] Events are past tense and immutable; handlers idempotent
- [ ] Subcontexts are not reachable by foreign contexts
- [ ] No Interface-tier component answers a question two audiences could answer
      differently (4.6)
- [ ] Authentication is in Transport; authorization is in the Domain tier
- [ ] The acting identity is an explicit parameter, never ambient
- [ ] Contract structure is framework-free; only the binding knows the protocol
- [ ] Configuration is read only at the composition root
- [ ] Technical exceptions are not used for business outcomes
- [ ] Naming follows section 7

---

## 12. Unsettled

Open questions. These are **not rules**. Pick per project and record the choice
in that project's decision log.

### DIMDAH-OPEN-1 — How a context consults a dedicated authorization context

When authorization grows complex enough to earn its own context, how do other
contexts consult it?

- **(a) The auth context decides; the calling Lobe asks its Port for a verdict.**
  The rule exists once. Costs a query per decision, and the verdict type must
  express every question callers need answered.
- **(b) The auth context exposes permission facts; each Lobe decides.** Fewer
  round trips, but the rule is re-derived per context — the divergence DIMDAh
  exists to prevent.
- **(c) The verdict is resolved once at the Interface tier and passed inward as
  a capability.** Fast and uniform, but puts a decision in the Interface tier
  unless the capability is issued by the auth context itself.

Leaning (a): it is the only option where the rule has one home.

### DIMDAH-OPEN-2 — How the contract's transport binding reaches clients

- **(a) Full SDK**, including a concrete HTTP client. Best when the whole stack
  shares one HTTP library; forces that library on every consumer.
- **(b) Interfaces only** — the client implements transport itself. Maximum
  neutrality, most client work.
- **(c) A transport compatibility layer** the client's existing framework binds
  to (tapir-style). Most flexible, most machinery to build and maintain.
