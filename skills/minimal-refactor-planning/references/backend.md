# Backend Refactor Heuristics

Use this reference when the problem is mostly in handlers, routes, services, jobs, workers, scripts, CLI commands, or multi-step server-side flow.

## Common backend smells

### 1. Handler/controller owns too much
Typical signal:

- request parsing
- validation
- authorization
- orchestration
- external API calls
- persistence
- response shaping

all mixed inside one function.

Preferred direction:

- keep protocol/request concerns near the edge
- move business flow into a service/use-case layer
- isolate external integrations behind explicit boundaries

### 2. Same business flow appears in multiple endpoints or jobs
Typical signal:

- similar logic copied across route handlers, cron jobs, consumers, or commands
- small drift has already started between copies

Preferred direction:

- centralize the business flow
- let each entrypoint adapt inputs/outputs only

### 3. Infrastructure leaks everywhere
Typical signal:

- database calls, queue publishing, HTTP clients, logging, and domain decisions interwoven
- impossible to test the core decision path without booting everything

Preferred direction:

- separate orchestration from infrastructure adapters
- define a narrower core path that can be tested with stubs/fakes

### 4. Hidden coupling through shared util sprawl
Typical signal:

- many “utils” with unclear ownership
- behavior depends on side effects or shared mutable state
- changes in one place unpredictably affect others

Preferred direction:

- group logic by responsibility, not by generic “util” naming
- create focused modules with clear contracts

### 5. Large refactor temptation
Typical signal:

- the code is messy, so the impulse is to redesign routes, services, repositories, and domain model at once

Preferred direction:

- refactor only around the path needed for the requested change
- improve the boundary that reduces duplication or confusion now
- stop once the requested change becomes straightforward

## Choosing the right abstraction

### Service / use-case
Use when the main need is to centralize business orchestration.

### Adapter / gateway
Use when isolating external systems:

- database
- third-party APIs
- queues
- storage
- RPC clients

### Helper / module
Use when extracted logic is deterministic and mostly pure.

### Command wrapper / job runner
Use when multiple operational entrypoints should share one execution path.

## Preferred answer shape for backend cases

Good backend plans usually clarify:

1. what stays at the transport edge
2. what moves into the service/use-case layer
3. what infrastructure boundary should be isolated
4. what behavior remains unchanged
5. how to validate without exploding scope

## Backend-specific trade-off language

Useful phrases:

- “Keep request/transport concerns at the edge and move the business flow inward.”
- “These endpoints should become thin adapters over one shared execution path.”
- “This isolates external I/O without forcing a broader architecture rewrite.”
- “For this change, a focused service extraction is enough.”

## Common pitfalls

Avoid recommending these by default:

- full domain-driven redesign for a local cleanup
- repository/service/factory layering with no immediate payoff
- premature abstraction around a single caller
- mixing protocol concerns with business semantics after the refactor
- renaming everything while also changing behavior
