# DIMDAh — Rationale and Worked Examples

**Non-normative.** Nothing here is a rule. [`dimdah.md`](./dimdah.md) is the
specification; this file argues why those rules exist and shows them applied.
Where the two appear to disagree, the specification wins.

---

## Part 1 — Why

### The cognitive load problem

The difference between a good architecture and a bad one is measured by **how
much of the codebase you must read and understand to safely change a small part
of it**. Traditional layering fails this test through global dependencies,
hidden coupling, unclear ownership of behavior, and abstractions whose changes
ripple unpredictably.

A good architecture minimizes the blast radius of *comprehension*: fixing a bug
should require understanding the immediate context, not the system.

### The AI collaboration problem

Modern coding agents deliver code well but lose coherence as a codebase grows
interconnected — not only because of context limits, but because reasoning
across a dense dependency graph is hard. When everything can touch everything:

- side effects and implicit coupling become unreasonable about,
- context fills with tangentially related code,
- suggestions get less precise,
- the cognitive burden shifts from human to machine without shrinking.

DIMDAh's boundaries are what let an agent load one context and be right about
it. This is also why the specification is written as rules rather than prose:
an agent can check a rule, and cannot check an essay.

### Why a tree, and why strictly upward

SET's single rule — a component is accessible only to its direct parent — buys
three things:

- **You can always predict where code goes.** Lower read and write cost.
- **Nobody can bypass a rule by accident.** An Orch cannot reach into a Repo and
  skip a Kernel invariant, because the compiler refuses.
- **Each layer is ignorant of its callers.** It can be read, understood and
  changed alone.

The Port/Gate exception exists because contexts must talk. Splitting queries
from transactions means "who may read me" and "who may change me" are separate,
compiler-visible questions.

### Why no layer collapsing (changed in 0.2)

Earlier versions allowed omitting a Lobe when it would only forward, treating
the absence as a signal that no higher-order logic existed. Two problems showed
up in practice:

1. The rule required a judgment call at every context's birth, and the reward
   for guessing wrong was a migration — the Kernel's visibility changes, Orch's
   calls change, tests move.
2. Two vendored copies of this document drifted on exactly this point, and
   codebases were written against both readings.

A forwarding Lobe costs two lines and makes every context the same shape.
Uniformity is worth more than the saved boilerplate — especially for an agent,
which pays for each judgment call with an opportunity to be wrong.

### Why duplication of types is free and duplication of logic is fatal

Two contexts holding their own `Address` cost a little typing. One shared rich
`Address` couples them: every change negotiates with every user, and the
coupling is invisible in the type.

A duplicated *rule* is the opposite. "Who may rename a user" implemented in an
admin API and again in a user API will eventually disagree, and the disagreement
is a security bug, not a style problem. This is the reasoning behind the
Interface tier's precise prohibition: not "no code", but no decision two
audiences could answer differently.

### Why errors are data

A signature that says `IO[RegistrationError, UserId]` tells a caller exactly
which business outcomes exist, and the compiler makes them handle each one. A
thrown exception says nothing, and a `String` error says nothing checkable. The
same argument extends to the wire: an HTTP client that parses a tagged error
body can dispatch exhaustively; one that reads a prose message cannot.

---

## Part 2 — Worked examples

### Error flow through the layers

Repo errors are storage-shaped and never escape the Kernel:

```scala
sealed trait DbError
case object RecordNotFound   extends DbError
case object UniqueConstraint extends DbError
```

The Kernel translates them into the aggregate's vocabulary:

```scala
private[x] def register(email: String, pw: String): IO[UserKernelError, UserId] =
  repo.insert(row).mapError {
    case UniqueConstraint => EmailAlreadyTaken
    case other            => InvalidState(other.toString)
  }
```

The Lobe adds decision-level errors, including those that came from querying a
foreign Port:

```scala
private[context] def registerWithPlan(email: String, pw: String, planId: PlanId)
    : IO[RegistrationError, UserId] =
  for {
    plan <- billingPort.getPlan(planId).mapError(_ => PlanNotAvailable)
    _    <- IO.ensure(plan.isActive, RegistrationClosed)
    id   <- kernel.register(email, pw).mapError {
              case UserKernelError.EmailAlreadyTaken => RegistrationError.EmailAlreadyTaken
            }
  } yield id
```

The Orch maps the Lobe's internal vocabulary to the public one. When the two
coincide the mapping is trivial — the boundary still exists, because the Port's
type is public and the Lobe's is not.

### Cross-context error composition

```scala
class CheckoutOrch(userPort: UserPort, paymentGate: PaymentGate) extends CheckoutGate {
  def checkout(userId: UserId, items: List[Item]): IO[CheckoutError, OrderId] =
    for {
      user    <- userPort.getProfile(userId).mapError(_ => CheckoutError.UserNotFound)
      _       <- paymentGate.charge(user.id, total).mapError(_ => CheckoutError.PaymentFailed)
      orderId <- orderGate.create(user.id, items).mapError(_ => CheckoutError.OrderRejected)
    } yield orderId
}
```

Every `mapError` is an explicit translation boundary; it is what keeps a
payment context's internal error out of a checkout consumer's match statement.

### A saga with compensations

```scala
def checkout(userId: UserId, items: List[Item]): IO[CheckoutError, OrderId] =
  for {
    _       <- inventoryGate.reserve(items)
    _       <- paymentGate.charge(userId, total)
                 .onError(_ => inventoryGate.release(items))
    orderId <- orderGate.create(userId, items)
                 .onError(_ => paymentGate.refund(userId, total) *> inventoryGate.release(items))
  } yield orderId
```

Note what is absent: no `if`, no validation, no derived business value. Apply
the litmus test from §3.4 and the method survives intact — it is genuinely
orchestration.

### Events instead of reaching in

```scala
// ❌ direct call into another context's internals
def updateProfile(id: UserId, data: ProfileData): IO[Error, Unit] =
  for {
    _ <- kernel.applyUpdate(id, data)
    _ <- cqrsService.updateReadModel(id, data)
  } yield ()

// ✅ emit a fact; interested contexts subscribe
def updateProfile(id: UserId, data: ProfileData): IO[Error, Unit] =
  for {
    _ <- kernel.applyUpdate(id, data)
    _ <- eventPublisher.emit(ProfileUpdated(id, data))
  } yield ()
```

For reliable delivery, write the event to an outbox inside the same transaction
as the state change, and let a separate process publish it.

### The Interface tier, end to end

One endpoint — "change username" — where an admin may rename anyone and a user
may rename only themselves.

**Contract (structure)** — one trait per audience, audience in the package:

```scala
// api/user/UserProfileApi.scala
trait UserProfileApi:
  def changeUsername(session: SessionToken, newUsername: String)
      : IO[Unauthorized.type | ValidationError | DuplicateUsername.type, Unit]
```

**Contract (binding)** — the same contract, one protocol down: codecs, the
status per error, path and verb. No Pekko, no Netty, no client library.

**Transport** — authenticates, deserializes, calls the Impl, serializes the
result or the tagged error with its status.

**Impl** — translation only:

```scala
class UserProfileApiImpl(identity: identity.Gate) extends UserProfileApi:
  def changeUsername(session: SessionToken, newUsername: String) =
    for
      actor <- authenticate(session)                       // session → actor
      _     <- identity.changeUsername(actor, actor.userId, Username(newUsername))
                 .mapError(toContractError)
    yield ()
```

**Domain** — the decision, once, for both audiences:

```scala
// identity/Gate.scala
def changeUsername(actor: Actor, target: UserId, name: Username)
    : IO[Forbidden.type | UsernameTaken.type | InvalidUsername, Unit]
```

The admin Impl differs only in which `target` it passes. Neither Impl knows the
permission rule; the Lobe behind the Gate does, and it is tested once.

### Testing, layer by layer

```scala
// Kernel — invariants, mocked Repo
test("kernel rejects a duplicate email") {
  when(repo.findByEmail("a@b.com")).thenReturn(IO.succeed(Some(existingRow)))
  assert(kernel.register("a@b.com", "hash").runSync() == Left(EmailAlreadyTaken))
}

// Lobe — decisions, mocked foreign Port
test("lobe refuses registration when the plan is inactive") {
  when(billingPort.getPlan(planId)).thenReturn(IO.succeed(inactivePlan))
  assert(lobe.registerWithPlan("a@b.com", "pw", planId).runSync() == Left(PlanNotAvailable))
}

// Orch — compensation paths, mocked Lobes and foreign Gates
test("checkout releases inventory when payment fails") {
  when(paymentGate.charge(any, any)).thenReturn(IO.fail(PaymentFailed))
  checkoutOrch.checkout(userId, items).runSync()
  verify(inventoryGate).release(items)
}
```

The pyramid follows from where decisions live: many Kernel tests, moderate Lobe
tests, few Orch tests, minimal Interface tests.

---

## Part 3 — Adoption

### When DIMDAh fits

Complex business domains with substantial logic; long-lived systems; small
teams that must move fast without breaking invariants; AI-assisted development,
where explicit boundaries are what keep an agent correct.

### When it is overkill

CRUD applications with little logic; prototypes with short lifespans; systems
whose performance constraints demand tight coupling.

### Order of work

1. Identify contexts — map business capabilities to bounded contexts.
2. Define Port and Gate first: what does this context let others read, and what
   transactions does it expose?
3. Implement Kernels — invariants and persistence.
4. Add the Lobe from day one (0.2 removed collapsing), even if it forwards.
5. Wire through Orch; introduce sagas as cross-context needs appear.
6. Turn on package-private visibility immediately — retrofitting it is the
   expensive path.

### Deployment

Context boundaries are logical, not physical. A system can start as a monolith
and extract contexts into services later, provided the boundaries were
respected; Ports and Gates are the natural service seams.
