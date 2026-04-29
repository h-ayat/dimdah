# DIMDAh — Interface Tier

> **Status**: Placeholder. This document will specify the Interface tier of DIMDAh.

## Overview

The Interface tier is the **outermost layer** of a DIMDAh system. It exposes the Domain tier to the external world and is the only place where protocol-specific concerns (HTTP, gRPC, messaging, CLI, etc.) are handled.

It sits above the Domain tier and has no knowledge of the Infrastructure tier's implementations:

```
Interface   ◄── external world enters here
    │
  Domain
    │
Infrastructure
```

---

## Planned Content

- Endpoint and controller structure
- DTO / View Model patterns and naming conventions
- Error-to-protocol translation (domain errors → HTTP status codes, gRPC status codes, etc.)
- Authentication and authorization at the boundary
- Input validation and sanitization (what belongs here vs in the Domain)
- Protocol handler patterns (HTTP middleware, gRPC interceptors, message consumers)
- Dependency wiring: how Interface components receive Domain Ports
- Testing strategy for the Interface tier (contract tests, mock Ports)
