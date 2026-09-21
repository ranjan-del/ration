# ADR 0003: Measurement lives inside the kernel, not in a separate harness

- Status: accepted
- Date: 2026-09-22

## Context

The original plan sequenced inference economics as the eighth research direction, after the context
runtime, routing, memory compilation and continual learning were built.

Two findings from the 2026 literature argue against that order. First, unified evaluation of model
routing reports that most methods collapse to similar performance and several fail to beat simple
baselines, meaning a method built without a neutral harness proves nothing. Second, long context
evaluation finds models effectively use only a small fraction of nominal context, meaning size based
claims are worthless and only measured quality against measured cost says anything.

Building the runtime first would produce a system with no credible way to show it beats the
combination of existing point solutions.

## Decision

The ledger and the evaluator are components of the kernel, present from phase 1, not a harness added
later. The kernel cannot execute a plan without recording it. Later phases consume the ledger as
input to their own decisions, which is only possible because it was there from the start.

Phase 1 therefore ships the measurement substrate **and** one real decision, so that it is a working
system rather than a benchmark suite.

## Consequences

Positive:

- Every phase after the first inherits the ability to prove or disprove itself.
- Phase 6 policy learning has training data as a byproduct of normal operation.
- The measurement substrate is independently useful to other people even if the rest of the
  programme fails, which is the most likely single positive outcome of this project.

Negative:

- Phase 1 is larger than a phase 1 usually is, and produces a smaller visible capability than
  building the context runtime first would have.
- Recording every run has a cost. It is measured and reported like anything else.

## Alternatives rejected

| Alternative | Why rejected |
|---|---|
| Build the context runtime first, measure later | The resulting claims would be unfalsifiable, which is the failure mode the landscape review identifies in the field |
| Build a standalone benchmark harness with no runtime | Would not be the system the agenda calls for, and nothing would consume the ledger |
