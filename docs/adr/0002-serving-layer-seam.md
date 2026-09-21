# ADR 0002: The seam between Ration and any serving layer is the Plan and Outcome contract

- Status: accepted
- Date: 2026-09-22

## Context

`loomrun` is a scheduler and governor for one shared inference machine. Its planned phases include a
ledger of caller, model, tokens, duration and outcome, per caller token and time budgets, and a
degradation ladder that falls back to a smaller model or a shorter context. It targets the same
hardware as Ration.

That overlaps substantially with Ration's compute policy and resource accounting. Left undefined, the
two projects would build two ledgers and two budget systems that disagree with each other.

## Decision

Keep the projects separate, with a declared seam.

| | Loomrun | Ration |
|---|---|---|
| Answers | Given a request, run it safely and fairly on this box | What request should exist in the first place |
| Layer | Data and control plane for one machine | Decision layer above any machine |
| Changes when | Hardware, concurrency, engines and failure modes change | Task shapes, budgets, information needs and models change |

The seam is the `Plan` and `Outcome` contract. Loomrun becomes one implementation of the `Executor`
port, alongside Ollama, MLX, a deterministic double and later vLLM.

Two constraints follow and are binding:

1. Ration must be fully usable without Loomrun. A research artifact that requires the author's other
   unfinished project cannot be run or compared by anybody else, which defeats its purpose.
2. Loomrun's budget governor enforces budgets it is handed. It does not decide them. Deciding a
   budget is a Ration concern.

## Consequences

Positive:

- One system of record for what a run cost, not two.
- Each project has one reason to change.
- Either project can be picked up, or abandoned, without stranding the other.

Negative:

- The `Outcome` contract has to be rich enough for both a research ledger and an operational ledger,
  which makes it larger than either would need alone.
- Loomrun's roadmap needs amending. Its phase 6 governor shrinks. That amendment is owed to that
  repository and is not done by this decision.

## Alternatives rejected

| Alternative | Why rejected |
|---|---|
| Ration depends on Loomrun | Loomrun has nothing shipped. A public research programme cannot be blocked on it |
| Ration absorbs Loomrun | Merges two systems with independent reasons to change, and discards a repository with a clear problem statement |
| Leave both independent and duplicate | Two ledgers, two budget models, guaranteed divergence |
