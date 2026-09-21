# Design: Ration, a policy kernel for quality per resource

- Date: 2026-09-22
- Status: approved
- Supersedes: nothing
- Related: [ADR 0001](../adr/0001-policy-kernel-over-pluggable-executors.md) through [ADR 0005](../adr/0005-plan-is-data.md)

This document records the design that came out of the design session on 2026-09-22, including the
approaches that were rejected and why. The living descriptions of the system are
[architecture.md](../architecture.md), [problem.md](../problem.md),
[evaluation.md](../evaluation.md) and [landscape.md](../landscape.md). This document is the record of
how they were arrived at, and should not be edited to stay current.

---

## 1. Origin

The programme began as a research agenda covering ten directions and seven phases, aimed at one
question: can AI systems become more capable by managing information, memory, computation and
reasoning far better, rather than by increasing model size and context length. That agenda is
preserved unedited in the `contextos` repository.

A literature review on 2026-09-22 found that nine of the ten directions are occupied by active,
well resourced work, and recorded the closest systems and the specific gap for each in
[landscape.md](../landscape.md). Two findings changed the plan.

| Finding | Consequence |
|---|---|
| Unified 2026 evaluations of model routing report that most methods collapse to similar performance and several fail to beat simple baselines | The scarce resource is not another method, it is trustworthy measurement. The same pattern is appearing in cache eviction and agent memory |
| Long context evaluation finds models effectively use only a small fraction of nominal context | Claims of the form "supports N tokens" are configurations, not results. Only quality measured against cost says anything |

## 2. Approaches considered

| | A. Policy kernel with measurement inside it | B. Context runtime first, as originally ordered | C. Memory first, paging treated as retrieval |
|---|---|---|---|
| Phase 1 delivers | A working decision loop making one real decision end to end, with ledger, evaluator and baselines inside the kernel | The memory hierarchy and demand paging machinery | The compiled memory representation and lifecycle |
| Honest on the primary hardware | Yes | No. Unified memory has no tier split to page across, so the central mechanism cannot be shown | Yes |
| Open surface against 2026 prior art | Good. No existing system decides tier, resolution, model and budget under one policy and one metric | Thin. LMCache, Mooncake, Quest and NVMe tiering own this in production | Weakest. The most crowded direction in the review |
| Keeps all ten directions reachable | Yes | Partly | Partly |
| Main risk | The kernel abstraction could be wrong early | Blocked on hardware that may never be available | Drifts away from the compute and economics half of the agenda |

**Chosen: A.** The deciding argument is that in A the measurement is a component of the system rather
than a separate phase. The ledger and evaluator are how the kernel decides, not only how it is
graded, so phase 1 is a working system and not a benchmark suite.

The main risk in A is addressed by [ADR 0005](../adr/0005-plan-is-data.md): making `Plan` versioned
data rather than code paths converts "the abstraction was wrong" from a rewrite into a migration.

The context runtime is not demoted by this choice. Phase 1 ships its central hypothesis in the
simplest honest form, which is that selecting the minimum sufficient context under a budget holds
quality. Phase 2 is the full context runtime, and it arrives with a ledger, four baselines and an
evaluator already in place.

## 3. The design

### 3.1 Kernel

One loop, described in [architecture.md](../architecture.md). Each phase widens what the planner is
permitted to decide and changes nothing else.

### 3.2 Ports

Six: `Executor`, `InformationStore`, `Selector`, `Evaluator`, `Meter`, `Ledger`. Each has at least
two implementations, one deterministic and offline.

The `Selector` port carries the fairness of the whole project. The four baselines are implementations
of it, so the system under test and the thing it claims to beat differ in exactly one object.

### 3.3 Contracts

`Plan` and `Outcome` are the only objects crossing the executor boundary, and they are the seam with
any serving layer. Field by field definitions are in [architecture.md](../architecture.md).

### 3.4 Measurement

Rules, baselines, metric and reproducibility requirements are in [evaluation.md](../evaluation.md).
The two that shape the code rather than the prose:

- The report generator reads the ledger, and there is no code path that writes a number by hand.
- The ledger refuses to compare records across hardware profiles.

### 3.5 Boundaries with other projects

| Project | Boundary | Recorded in |
|---|---|---|
| `contextos` | The agenda this implements. Documents only | This document, section 1 |
| `loomrun` | Serving layer. One `Executor` adapter. Ration never requires it. Its governor enforces budgets rather than deciding them | [ADR 0002](../adr/0002-serving-layer-seam.md) |
| `ragfabric` | Retrieval platform. Optionally driven through `InformationStore`. No dependency | README |

## 4. Phases

Seven, listed with exit criteria in [ROADMAP.md](../../ROADMAP.md). The binding rule is that every
phase ships a component in its final shape. If a phase would produce something a later phase throws
away, the phase is redesigned rather than shipped.

Phase 5 is scheduled late deliberately. It is the most crowded direction in the landscape review and
the one where novelty is least likely. Scheduling it early would spend the project's best energy in
the worst place.

Phase 4 has a publishable negative result available to it, and the roadmap says so. If routing on
appraised difficulty does not beat a simple baseline on this harness, that finding is reported as the
phase 4 result rather than being tuned away.

## 5. Scope of the first session

Scaffold and documents only. No implementation code. The `packages/` tree is described by the phase 1
plan and does not exist yet.

## 6. Open questions

Carried forward rather than resolved, and tracked in the local memory file.

| Question | Blocks | Notes |
|---|---|---|
| Which task suites, and the reason for each | Phase 1 exit criterion | Candidates and what each falsifies are listed in [evaluation.md](../evaluation.md). The choice needs to be made before the selector is written, because a suite chosen after seeing results is not evidence |
| What the sufficiency signal actually is | Phase 3, and the `Verdict` contract | Options range from asking the model directly to inferring from answer confidence to a separate verifier. Each has a different cost, and the cost is part of the result |
| How the oracle selector is constructed per task family | Phase 1 | An oracle that is too generous makes every method look bad, and one that is too weak makes every method look good |
| Whether energy is measurable well enough on the primary platform to report | Phase 1 meter | If not, the field stays null under ADR 0004 |
| Amending the Loomrun roadmap for the seam in ADR 0002 | Nothing here | Owed to that repository, not done by this decision |

## 7. Self review

Checked on 2026-09-22 against the four criteria this process requires.

| Criterion | Result |
|---|---|
| Placeholders | None. Section 6 records genuinely open questions as open questions, with what each one blocks, rather than leaving gaps in the design |
| Internal consistency | The phase table, the contract table and the roadmap were cross checked. The architecture matches the phase descriptions |
| Scope | Phase 1 is a single implementation plan. Phases 2 through 7 each need their own design document before implementation, and this document does not attempt to be one |
| Ambiguity | "Minimum sufficient context" is defined operationally as the selection that maximises quality per materialised token on a task family, measured against the oracle, rather than left as a phrase |
