# Frontend Refactor Heuristics

Use this reference when the problem is mostly in UI code, page orchestration, state ownership, component responsibilities, or user interaction flow.

## Common frontend smells

### 1. Multiple components each implement the same orchestration
Typical signal:

- several buttons / menus / empty states trigger the same async chain
- each component owns slightly different copies of the flow

Preferred direction:

- move the real flow into one shared owner
- keep each UI surface as a trigger only

### 2. Presentation and side effects are mixed together
Typical signal:

- one component both renders UI and manages file handling / requests / data shaping / success toasts / retries

Preferred direction:

- keep rendering close to the component
- extract orchestration into a hook/controller/shared module

### 3. State ownership is unclear
Typical signal:

- one state value affects multiple sibling components
- callbacks are bouncing through several layers without a clear owner
- a child effectively owns behavior for unrelated peers

Preferred direction:

- move ownership to the nearest common parent
- pass down narrow props and explicit handlers

### 4. Indirect communication
Typical signal:

- CustomEvent
- window events
- ad-hoc global signals
- refs used as a hidden command bus

Preferred direction:

- explicit prop callback
- shared hook owned by parent
- stable controller/service boundary

Use indirect communication only when the tree boundary truly requires it.

### 5. Overlifting state
Typical signal:

- everything is moved to a page/global store even though the behavior is local

Preferred direction:

- lift only to the nearest level that actually needs coordination
- avoid store/globalization unless multiple distant consumers truly need it

## Choosing the right abstraction

### Shared helper
Use when the extracted logic is mostly pure:

- parse
- transform
- map
- validate
- derive payloads

### Hook
Use when the flow is interactive and stateful:

- pending state
- selection state
- retries
- error/success UI behavior
- imperative trigger for UI actions

### Page-level owner
Use when multiple sibling components need the same capability or shared state.

### Service / uploader / manager
Use when the workflow centers on asynchronous orchestration, remote calls, files, or resources and should not live inside a visual component.

## Preferred answer shape for frontend cases

Good frontend plans usually clarify:

1. which layer owns the state
2. which component becomes trigger-only
3. what logic gets centralized
4. what communication mechanism is removed
5. what regression path must be retested

## Frontend-specific trade-off language

Useful phrases:

- “This keeps the UI surface thin and moves orchestration into one owner.”
- “The empty state and the menu become two entry points into the same flow.”
- “This removes event-based coordination in favor of an explicit parent-owned trigger.”
- “This is enough for this change; splitting further would be premature.”

## Common pitfalls

Avoid recommending these by default:

- introducing global state for page-local behavior
- extracting many tiny hooks before the boundary is proven
- keeping duplicated async logic in separate components
- solving ownership problems with events instead of hierarchy
- forcing a reusable framework abstraction for a one-off workflow
