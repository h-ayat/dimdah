# DIMDAh — Error Handling Guide

## Overview

DIMDAh makes a strict distinction between two fundamentally different kinds of failures:

- **Domain errors** — expected outcomes that represent business rule violations (e.g., email already taken, insufficient balance, plan not found). These are part of the contract.
- **Technical exceptions** — unexpected infrastructure failures (e.g., DB connection lost, network timeout, out of memory), or unexpected inconsistencies. These are not part of the business contract.

Confusing the two leads to unclear APIs, swallowed errors, and callers that cannot distinguish "the business said no" from "something broke."

---

## Core Pattern: `IO[E, A]`

All business logic functions return a value in an effect type that encodes both dimensions:

```
IO[E, A]
  E = domain error type  (business rule violation — caller must handle)
  A = success type
  technical exceptions   (wrapped by the effect system — separate channel)
```

Example:
```scala
def registerUser(data: UserData): IO[UserRegistrationError, UserId]
```

This signature communicates exactly:
- On success: returns a `UserId`
- On domain failure: returns a `UserRegistrationError` (caller must handle all cases)
- On infrastructure failure: the effect system raises a defect/exception (not in the type)

By encoding domain errors in the type signature, invalid states are visible at compile time and callers are forced to handle all logical outcomes.

### Implementation Options

`IO[E, A]` is a semantic contract, not a prescribed type. Use whatever fits your stack:

| Platform | Natural fit |
|----------|-------------|
| ZIO (Scala) | `ZIO[Any, E, A]` |
| Cats Effect (Scala) | `EitherT[IO, E, A]` or custom |
| Arrow (Kotlin) | `Either<E, A>` in a coroutine |
| Rust | `Result<A, E>` |
| Custom | `Future[Either[E, A]]` with helpers |

The key requirement: **technical failures and domain errors must be distinguishable** — by the type, the channel, or an explicit wrapper.

---

## Error Types as ADTs

Domain error types are **sealed ADTs** (algebraic data types). This forces callers to handle every case at compile time:

```scala
sealed trait UserRegistrationError
case object EmailAlreadyTaken      extends UserRegistrationError
case object InvalidEmailFormat     extends UserRegistrationError
case class  WeakPassword(reason: String) extends UserRegistrationError

// Caller must handle all cases — the compiler will warn on missing branches
def handleError(err: UserRegistrationError): Response = err match {
  case EmailAlreadyTaken       => Response.conflict("Email in use")
  case InvalidEmailFormat      => Response.badRequest("Invalid email")
  case WeakPassword(reason)    => Response.badRequest(s"Password too weak: $reason")
}
```

Never use `String` or `Exception` subtypes for domain errors. The goal is **exhaustive, typed dispatch**.

---

## Error Flow Through SET Layers

Each layer in the Domain tier has its own error vocabulary. Errors are translated at layer boundaries — inner errors are not blindly propagated up.

### Repo Errors

Repos interact with storage. Their errors are technical or storage-constraint violations:

```scala
sealed trait DbError
case object RecordNotFound   extends DbError
case object UniqueConstraint extends DbError
case object ConnectionFailed extends DbError  // usually a defect, not domain error
```

Repo errors are **package-private** and never exposed outside the kernel. The kernel translates them into domain errors.

### Kernel Errors

Kernels translate raw Repo errors into domain errors and add their own invariant violations:

```scala
sealed trait UserKernelError
case object EmailAlreadyTaken extends UserKernelError   // translated from UniqueConstraint
case object UserNotFound      extends UserKernelError   // translated from RecordNotFound
case class  InvalidState(msg: String) extends UserKernelError

// Translation happens inside the kernel:
private def register(email: String, pw: String): IO[UserKernelError, UserId] =
  repo.insert(row).mapError {
    case UniqueConstraint => EmailAlreadyTaken
    case other            => InvalidState(other.toString)
  }
```

Kernel errors are the lowest-level domain errors and are **package-private to the lobe**. The lobe may enrich or re-wrap them.

### Lobe Errors

Lobes combine kernel errors with their own decision-level errors. They may:
- Re-export kernel errors directly (if the Lobe API is thin)
- Wrap kernel errors in a richer lobe-level error type
- Add their own error cases for failed business decisions

```scala
sealed trait RegistrationError
case object EmailAlreadyTaken  extends RegistrationError  // re-exported from kernel
case object PlanNotAvailable   extends RegistrationError  // lobe-level (from external port query)
case object RegistrationClosed extends RegistrationError  // lobe-level business rule

private def registerWithPlan(...): IO[RegistrationError, UserId] =
  for {
    plan <- billingPort.getPlan(planId).mapError(_ => PlanNotAvailable)
    _    <- ZIO.when(!plan.isActive)(ZIO.fail(RegistrationClosed))
    id   <- kernel.register(email, pw).mapError {
              case UserKernelError.EmailAlreadyTaken => RegistrationError.EmailAlreadyTaken
            }
  } yield id
```

Lobe errors are **package-private to the context** and form the internal vocabulary used by Orch.

### Orch / Port Errors

The Port defines the **public error vocabulary** of a context. Orch maps lobe errors to Port errors. Port errors are what consumers of the context depend on.

```scala
// Port.scala — public
sealed trait UserError
case object EmailAlreadyTaken  extends UserError
case object PlanNotAvailable   extends UserError
case object RegistrationClosed extends UserError

trait UserPort {
  def registerWithPlan(...): IO[UserError, UserId]
}

// Orch.scala
class UserOrch(lobe: UserLobe) extends UserPort {
  def registerWithPlan(...): IO[UserError, UserId] =
    lobe.registerWithPlan(...).mapError {
      case RegistrationError.EmailAlreadyTaken  => UserError.EmailAlreadyTaken
      case RegistrationError.PlanNotAvailable   => UserError.PlanNotAvailable
      case RegistrationError.RegistrationClosed => UserError.RegistrationClosed
    }
}
```

If Lobe and Port error types are the same (as often happens), the `mapError` is trivial or absent. But the boundary still exists — the Port type is public, the Lobe type is internal.

---

## Cross-Context Error Composition

When Orch coordinates multiple contexts, it must compose their errors into a unified error type:

```scala
sealed trait CheckoutError
case object UserNotFound         extends CheckoutError
case object PaymentFailed        extends CheckoutError
case object InventoryUnavailable extends CheckoutError

class CheckoutOrch(userPort: UserPort, paymentPort: PaymentPort) extends CheckoutPort {
  def checkout(userId: UserId, items: List[Item]): IO[CheckoutError, OrderId] =
    for {
      user <- userPort.getProfile(userId)
                .mapError(_ => CheckoutError.UserNotFound)
      _    <- paymentPort.charge(user, total)
                .mapError(_ => CheckoutError.PaymentFailed)
      // ...
    } yield orderId
}
```

Each `.mapError` call is an **explicit translation boundary** — it documents which external error maps to which checkout-level error and prevents internal error types from leaking across context boundaries.

---

## Interface Tier: Error-to-Protocol Translation

The Interface tier (HTTP, gRPC, etc.) is responsible for translating domain errors into protocol-appropriate responses. It is the **only** place where domain errors are converted to HTTP status codes, gRPC status codes, or error response bodies.

```scala
// HTTP endpoint
def registerRoute: Route =
  post("/users/register") { req =>
    userPort.registerWithPlan(req.email, req.password, req.planId).fold(
      error => error match {
        case UserError.EmailAlreadyTaken  => Response(409, "Email already in use")
        case UserError.PlanNotAvailable   => Response(402, "Selected plan is unavailable")
        case UserError.RegistrationClosed => Response(403, "Registration is currently closed")
      },
      userId => Response(201, s"/users/${userId.value}")
    )
  }
```

**Rule**: Never map domain errors to protocol responses inside Domain tier code. That translation belongs entirely in the Interface tier.

---

## Technical Exceptions vs Domain Errors

| | Domain Error | Technical Exception |
|--|-------------|---------------------|
| **Nature** | Expected business outcome | Unexpected infrastructure failure |
| **In signature** | Yes — `IO[E, A]` | No — defect channel |
| **Caller must handle** | Yes (compile-time) | Via global handler / middleware |
| **Examples** | `EmailAlreadyTaken`, `InsufficientBalance` | DB connection lost, OOM, network timeout |
| **Response** | Business-meaningful (409, 402, etc.) | Generic 500 / retry |

Technical exceptions should be caught at the **outermost boundary** (Interface tier or application entry point) and converted to generic failure responses. They must never be used to represent business logic outcomes.

---

## Common Anti-Patterns

**Using exceptions for domain errors**
```scala
// ❌ Wrong: throws instead of returning typed error
def register(email: String): UserId =
  if (emailExists(email)) throw new RuntimeException("Email taken")
  else createUser(email)

// ✅ Correct: typed error in signature
def register(email: String): IO[EmailAlreadyTaken.type, UserId] = ...
```

**Using a generic error type for all errors**
```scala
// ❌ Wrong: callers cannot exhaust cases; error meaning is unclear
def register(email: String): IO[String, UserId] = ...
def register(email: String): IO[Throwable, UserId] = ...

// ✅ Correct: specific ADT
def register(email: String): IO[UserRegistrationError, UserId] = ...
```

**Leaking inner layer error types**
```scala
// ❌ Wrong: Repo/Kernel error types exposed through Port
trait UserPort {
  def getUser(id: UserId): IO[DbError, User]  // DbError is an infrastructure type
}

// ✅ Correct: Port defines its own public error type
trait UserPort {
  def getUser(id: UserId): IO[UserNotFound.type, UserProfile]
}
```

**Swallowing errors**
```scala
// ❌ Wrong: error information lost
def register(email: String): IO[Nothing, Option[UserId]] =
  kernel.register(email).option  // None could mean "failed" or "not found"

// ✅ Correct: errors explicit
def register(email: String): IO[UserRegistrationError, UserId] =
  kernel.register(email)
```

---

## Checklist

- [ ] All domain logic functions use `IO[E, A]` (or equivalent typed error)
- [ ] Error types are sealed ADTs — not strings, not generic exceptions
- [ ] Inner layer errors (Repo, Kernel) are translated at each boundary — not propagated raw
- [ ] Port error types are public; Lobe/Kernel error types are package-private
- [ ] Interface tier handles error-to-protocol translation exclusively
- [ ] Technical exceptions are not used for business outcomes
- [ ] Callers handle all error cases (compiler enforces via exhaustive match)
