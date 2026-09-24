# DIMDAh — Domain-Isolated Modular Driven Architecture

An architectural pattern that minimizes cognitive load in complex systems
through enforced isolation, strong typing and compiler-checked boundaries — so
that a developer, or an AI agent, can change one part of a system correctly
without understanding all of it.

## Documents

| File | Status | Contents |
|------|--------|----------|
| [`dimdah.md`](./dimdah.md) | **Normative** | The specification: tiers, SET, the four Domain layers, the Interface tier's three sub-layers, Infrastructure, error handling, naming, testing, anti-patterns, review checklist, open questions |
| [`rationale.md`](./rationale.md) | Non-normative | Why the rules exist, and worked examples applying them |

Every rule lives in `dimdah.md` and nowhere else. If `rationale.md` and the
specification appear to disagree, the specification wins.

## Key principles

- **Minimize cognitive load** — change one part without understanding the whole
- **Compiler over convention** — visibility modifiers and typed errors, not documentation
- **Domain errors are data** — expected failures appear in signatures
- **Duplicate types freely; never duplicate a decision**
- **Events for cross-context side effects**

## Using it in a project

Vendor `dimdah.md` into the project's documentation, unmodified, and record the
version adopted. Project-specific choices — the effect type, the module layout,
and the answers to the open questions in §12 — belong in that project's own
decision log, never in the vendored copy. Update by replacing the file wholesale
and bumping the recorded version.

The reference language is Scala 3; the visibility rules assume it. Other
languages apply the same rules with their own enforcement mechanism.

## Versioning

`dimdah.md` carries a version line under its title, bumped on every normative
change and tagged in this repository. `head -3` on a project's copy shows which
version it is on.

- **0.2** — Interface and Infrastructure tiers specified; error handling folded
  into the specification; layer collapsing removed; Orch-vs-Lobe litmus test;
  Port and Gate in separate files; prose split out into `rationale.md`.
- **0.1** — Domain tier only.

## Status

An evolving specification. Section 12 of the specification lists what is
deliberately unsettled.
