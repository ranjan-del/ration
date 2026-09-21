# Roadmap

Design specification: [docs/design/2026-09-22-ration-design.md](docs/design/2026-09-22-ration-design.md).
Primary hardware: Apple silicon with unified memory. Other hardware when it is free.
All numbers are labelled with the hardware profile that produced them and are never compared across profiles.

| Release | Phases | Headline |
|---|---|---|
| v0.1.0 | 1 | Kernel, ledger, baselines, and minimum sufficient context under a budget |
| v0.2.0 | 2 | Virtual context: pages, resolutions, faults, prefetch |
| v0.3.0 | 3 | Reversible context |
| v0.4.0 | 4 | Compute policy: routing and reasoning budget |
| v0.5.0 | 5 | Memory compiler and memory evolution |
| v0.6.0 | 6 | Policy learned from the ledger |
| v1.0.0 | 7 plus hardening | One joint budget optimiser over every decision |

Legend: `[x]` shipped on main, `[~]` in progress, `[ ]` not started.

Every phase ends with a measured before and after result recorded under `docs/benchmarks/`,
generated from the ledger, never written by hand.

Every phase ships a component in its final shape. If a phase would produce something a later phase
has to throw away, the phase is wrong and gets redesigned rather than shipped.

---

## Phase 0: Landscape and problem statement

- [x] `docs/problem.md`: the problem, the non goals, and what would falsify the thesis
- [x] `docs/landscape.md`: prior art per direction, with the specific measurable gap for each
- [x] `docs/architecture.md`: the kernel loop, the ports, the contracts
- [x] `docs/evaluation.md`: how anything in this repository is allowed to be measured
- [x] ADRs 0001 to 0004: kernel shape, the serving seam, measurement placement, hardware profiles
- [ ] Task suites chosen and written down, with the reason each one was chosen

## Phase 1: Kernel and ledger

The kernel gains one decision: what is the minimum sufficient context for this task under a token budget.

- [ ] `Plan`, `Outcome`, `Verdict`, `TaskProfile` and `HardwareProfile` as versioned data types
- [ ] Ports defined with no implementation detail leaking into `ration-core`
- [ ] Executor adapters: deterministic offline double, Ollama, MLX
- [ ] Meter for Apple silicon: wall time, time to first token, unified memory high water mark, tokens
- [ ] Ledger in SQLite, with the cross profile comparison refusal enforced in code
- [ ] Evaluator with a deterministic offline grader and at least one task suite grader
- [ ] Four baselines as first class code: full context, fixed truncation, plain retrieval, oracle context
- [ ] Selector: the first real Ration policy, choosing a context subset under a token budget
- [ ] `ration run`, `ration compare`, `ration report`
- [ ] Reproducibility: seed control, pinned model ids and quantisation, captured environment
- [ ] Measured: quality versus materialised context frontier against all four baselines, with the oracle gap

Exit criterion: `ration compare` produces a frontier plot against all four baselines on the primary
machine, reproducible from a seed, with the oracle gap reported, and the whole run reconstructable
from the ledger alone.

## Phase 2: Virtual context

The kernel gains: which page, at which resolution, in which tier, and when to fault, promote, demote,
evict and prefetch.

- [ ] Semantic pages and the multi resolution store
- [ ] The resolution ladder: raw tokens, detailed, summary, facts and entities and events, semantic
- [ ] Context faults: the model signals insufficiency, the runtime responds
- [ ] Promotion, demotion, eviction and prefetch policies
- [ ] Tier placement as an optional adapter, active only on hardware that has separate tiers
- [ ] Measured: quality held within a stated tolerance of full context while materialising a measured
      fraction of it, across a stated virtual size, on at least two task families

Exit criterion: the above measurement, plus an honest statement of which part of the phase could not
be demonstrated on the primary hardware and why.

## Phase 3: Reversible context

The kernel gains: when compressed is not enough, and what to expand.

- [ ] Compression with a recorded path back to the original
- [ ] Insufficiency detection that triggers expansion of a specific region rather than everything
- [ ] Measured: quality recovered versus the cost of recovering it, against never compressing

## Phase 4: Compute policy

The kernel gains: which model, how much reasoning, under a joint budget.

- [ ] Task difficulty appraisal with calibrated confidence
- [ ] Model routing and cascades across at least three model sizes
- [ ] Reasoning budget selection
- [ ] Degradation ladder: smaller model, shorter context, partial result with a stated reason
- [ ] Measured: against the strongest simple baseline on this harness

Exit criterion: a stated margin over the strongest simple baseline, or an honest report that there is
none. Published 2026 unified evaluations find that most routing methods fail to beat simple
baselines, so a negative result here is a real finding and gets published as one.

## Phase 5: Memory compiler and memory evolution

The kernel gains: what experience becomes, and how it ages, contradicts and consolidates.

- [ ] Compilation of raw experience into facts, concepts, entities, relationships, events, temporal
      states, rules, preferences, summaries, indexes and provenance, with the original recoverable
- [ ] Lifecycle states from new through confirmed and consolidated to archived
- [ ] Contradiction resolution with time awareness, so a superseded preference is not treated as current
- [ ] Decay, importance and forgetting
- [ ] Measured: on LongMemEval and BEAM, against published numbers, with the hardware caveat stated

## Phase 6: Learning and self evaluation

The kernel gains: which plans work for which task shapes, learned from its own ledger.

- [ ] Verification loop: answer, evaluation, evidence check, tool or test verification, confidence,
      failure diagnosis
- [ ] Policy learned from past outcomes. Policy learning over the ledger, not fine tuning the model
- [ ] Measured: learned policy against the fixed policy on held out tasks

## Phase 7: Runtime

- [ ] Every decision under one joint budget optimiser
- [ ] The full component set: context controller, memory manager, memory compiler, memory evolution,
      model router, compute budgeter, reasoning controller, cache manager, evaluation engine,
      learning controller
- [ ] Each component independently benchmarked, and the whole benchmarked against the best
      combination of point solutions tuned separately

## v1.0.0 hardening

- [ ] End to end tests, input validation, release automation, documentation site
- [ ] Reproduction package: anyone can rerun every published number from the repository

## Later, optional

- CUDA meter and vLLM executor adapter, once a CUDA machine is available
- Loomrun executor adapter
- Energy measurement beyond what the primary platform exposes
