---
name: minimal-refactor-planning
description: Provide a minimal, implementation-oriented refactor plan before writing code. Use when the user asks for the smallest-change design, wants options compared, needs responsibilities/state/logic boundaries clarified, or asks for a concrete implementation plan covering recommended approach, flow, affected modules, and validation. Works across frontend, backend, scripts, CLIs, and service code.
---

# Minimal Refactor Planning

Use this skill when the user does **not** want immediate code changes, but instead wants a practical implementation plan for modifying existing code with minimal disruption.

## Use this skill when

Trigger on requests like:

- "先别写代码，先给方案"
- "给我最小改法"
- "比较一下这几种改法"
- "这块应该怎么拆"
- "帮我梳理下实现方案"
- "给个落地计划"
- "尽量少动现有代码，怎么改"

Typical situations:

- responsibilities are mixed together
- logic is duplicated across multiple entry points
- state ownership is unclear
- modules/components depend on each other in awkward ways
- temporary mechanisms (events, glue code, implicit contracts) need cleanup
- the user wants a recommendation, not a vague architecture lecture

## Goal

Produce a plan that is:

1. minimal in change scope
2. explicit about trade-offs
3. concrete enough to implement next
4. careful about regressions
5. free of unnecessary abstraction

Default bias:

- prefer the smallest workable change
- prefer explicit dependencies over indirect wiring
- prefer consolidating shared logic into one place
- avoid platform-level redesign unless the user asks for it

## Default workflow

Unless the user asks otherwise:

1. Identify the real responsibilities currently mixed together
2. Find the narrowest boundary that can absorb the shared logic
3. Compare 2–3 viable approaches
4. Recommend one approach clearly
5. Describe the flow and affected areas
6. Define a small validation plan
7. Do **not** write code yet

## Default answer structure

### 1. Conclusion first

Start with the recommendation in one or two sentences.

Good examples:

- I’d use a minimal split here: move the shared flow into one owning layer, keep each entry point as a trigger only.
- I would not over-split this yet; one shared abstraction is enough for this change.

### 2. Option comparison

List 2–3 options. For each option, briefly cover:

- where the logic would live
- who owns core state / flow
- whether dependencies become cleaner or messier
- whether it fits this change’s scope

Do not present all options as equal. The recommendation should be obvious.

### 3. Recommended design

Describe:

- what abstraction to introduce, if any  
  (e.g. shared module, helper, service, manager, hook, controller, adapter)
- where it should live
- what it owns
- what it should **not** own
- how surrounding modules call into it

### 4. Flow

Describe the main execution path step by step:

1. where the flow starts
2. who receives it
3. who performs the core work
4. how success is propagated
5. how failure is handled
6. what existing behavior remains unchanged

Use “data flow”, “call flow”, or “execution flow” depending on the problem.

### 5. Change scope

List the practical edits:

- what logic moves
- what becomes a thin trigger/caller
- what new shared unit is added
- what temporary communication should be removed
- what interfaces/params need to change

### 6. Validation

Keep validation proportional. Usually include only:

- pure-function or helper verification where relevant
- targeted tests
- local lint / typecheck / build checks
- key regression paths

Do not silently expand the scope into a full rewrite or full test overhaul.

## Decision heuristics

### Prefer consolidating the real shared flow

If multiple entry points eventually do the same thing, centralize the actual handling. Let the edge locations keep only:

- triggering
- passing parameters
- rendering state or results

Avoid duplicating orchestration across many places.

### Prefer explicit dependencies

If the current design relies on:

- global events
- implicit contracts
- reversed ownership
- scattered glue code
- hard-to-follow cross-calls

Prefer:

- explicit injection
- direct method/callback boundaries
- a shared owning abstraction
- a straighter call graph

### Prefer one new abstraction over many

Do not split aggressively by default.

Add more layers only when at least one is clearly true:

- the logic is already reused across many places
- the behavior is evolving in multiple directions
- independent testing genuinely needs isolated boundaries
- one abstraction would otherwise own unrelated concerns

### Preserve behavior first

Default assumption: refactor should preserve semantics unless the user asked for behavioral change.

Protect:

- existing business meaning
- success/failure behavior
- callback timing
- return shape / output expectations
- core user-visible flow

### Avoid overdesign

Call out when something is theoretically elegant but not worth it for this change.

Common warning signs:

- too many new abstractions
- paying for future flexibility too early
- introducing infrastructure for a local problem
- broadening scope beyond the user’s ask

## Style requirements

- conclusion first
- practical, not academic
- explain trade-offs, not just structure
- optimize for low-risk execution
- do not dump the decision back on the user without a recommendation
- do not start coding unless requested

## Reusable response skeleton

Use this shape unless the user wants a different format:

- My recommendation:
- Option comparison:
  1. ...
  2. ...
  3. ...
- Recommended design:
- Main flow:
- Change scope:
- Validation:

## Common abstraction guide

When choosing where logic should go, use the lightest abstraction that fits:

- **helper / shared module**  
  for reusable pure logic

- **service / manager**  
  for orchestration, I/O, resource handling, multi-step flow

- **hook / controller**  
  for stateful interactive flow

- **adapter / bridge**  
  for compatibility boundaries or isolating third-party/legacy interfaces

Choose based on the problem, not by habit.

## References

Read these only when relevant:

- For frontend-oriented heuristics and common UI/state refactor patterns: `references/frontend.md`
- For backend/service/handler/job refactor patterns: `references/backend.md`
- For concrete answer shapes and example outputs: `references/output-examples.md`

## What not to do

Avoid responses that:

- stay abstract and never map to real edits
- list options without a recommendation
- propose a large redesign for a local issue
- optimize for elegance over change cost
- invent a future platform roadmap the user did not ask for
- sound comprehensive but do not guide implementation

The goal is not to sound architectural. The goal is to produce the next good move.
