# ADR 0005: Plan is versioned data, and phases add fields rather than code paths

- Status: accepted
- Date: 2026-09-22

## Context

The programme has seven phases, each adding a decision the kernel is allowed to make. The obvious
implementation, a branch in the kernel per capability, produces a kernel that is unreadable by phase
four and untestable by phase five. It also makes comparison across phases impossible, because a plan
from phase two cannot be expressed or replayed once phase four exists.

The acknowledged risk in choosing this architecture was that the kernel abstraction could be wrong
early, before enough is known to get it right.

## Decision

`Plan` is a versioned data object. A phase adds fields to it. The kernel gains no conditional
branches per phase. A planner that does not set a field leaves it at its documented default, and an
executor that does not understand a field rejects the plan rather than ignoring it.

`Outcome`, `Verdict`, `TaskProfile` and `HardwareProfile` follow the same rule.

## Consequences

Positive:

- Any plan is serialisable, replayable, diffable against another plan and executable by any executor
  that supports its version.
- A phase two policy and a phase four policy can be compared directly, because both produce the same
  kind of object.
- The wrong abstraction is recoverable: fields are added and deprecated, which is a migration, rather
  than the kernel being rewritten.

Negative:

- Contract versioning and compatibility rules have to exist from phase 1, which is work that buys
  nothing visible in phase 1.
- Rejecting unknown fields rather than ignoring them produces failures that ignoring would have
  hidden. That is intended.
