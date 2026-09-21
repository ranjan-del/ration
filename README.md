<h1 align="center">Ration</h1>

<p align="center">
  <b>A policy kernel that decides how much of anything an AI task actually needs.</b><br/>
  How much context, at what resolution, from which memory tier, on which model, with how much
  reasoning, under a budget you set. Every decision is recorded with what it cost and what it was
  worth, so the runtime can be judged instead of believed.
</p>

<p align="center">
  <a href="LICENSE"><img alt="License: Apache 2.0" src="https://img.shields.io/badge/license-Apache%202.0-blue.svg"></a>
  <img alt="Status: pre-alpha" src="https://img.shields.io/badge/status-pre--alpha-orange">
</p>

> **Status: planning.** Nothing is implemented yet. This README describes the problem, the
> architecture and the phase plan. [ROADMAP.md](ROADMAP.md) tracks what exists and what is next.
> No section below claims a feature that is not marked shipped in the roadmap, and no number will
> ever appear in this repository unless it came from a run recorded in the ledger.
> The long horizon research agenda this project serves is kept separately in `contextos`.
> Documentation lives in [docs/](docs/README.md).

---

## Table of contents

1. [The question](#the-question)
2. [What is actually wrong today](#what-is-actually-wrong-today)
3. [What Ration is](#what-ration-is)
4. [What Ration is not](#what-ration-is-not)
5. [The kernel loop](#the-kernel-loop)
6. [Ports](#ports)
7. [The contracts](#the-contracts)
8. [Honesty rules](#honesty-rules)
9. [How quality per resource is reported](#how-quality-per-resource-is-reported)
10. [Phases](#phases)
11. [Hardware](#hardware)
12. [Relationship to other projects](#relationship-to-other-projects)
13. [Prior art](#prior-art)
14. [Repository layout](#repository-layout)
15. [Contributing](#contributing)
16. [License](#license)

---

## The question

Can AI systems become more capable by managing information, memory, computation and reasoning
far better, rather than by continuously increasing model size and context length?

The objective is one ratio:

**useful answer quality per unit of context, computation, memory, latency, energy and money.**

Ration exists to make that ratio measurable first, and then to move it.

---

## What is actually wrong today

Four problems, each one observable rather than asserted.

| Problem | What it looks like in practice |
|---|---|
| Nominal context is not effective context | Models advertise very large context windows, but published long context evaluation finds they effectively use only a small fraction of it. Filling the window is not the same as using it |
| Every decision is made at maximum | The largest model, the full retrieved set, the maximum reasoning budget, for every request, whether the task needs it or not. The cost is paid whether or not it bought anything |
| Point solutions are benchmarked in isolation | KV cache eviction, retrieval, routing, compression and agent memory are each strong fields, and each system is measured on the benchmark its authors chose. Nothing measures them together under one budget |
| Methods collapse under fair evaluation | Unified evaluations of model routing published in 2026 find that most methods perform similarly and several fail to beat simple baselines. The same pattern is appearing in cache eviction and agent memory. The scarce thing is not another method, it is a neutral way to tell whether a method works |

Ration treats the fourth problem as the first one to solve, because the other three cannot be
honestly attacked without it.

---

## What Ration is

A kernel that sits between an application and whatever runs the model, and makes one decision per
request: given this task and this budget, what is the smallest amount of everything that will still
produce a good answer?

It answers by producing a **Plan**, handing that plan to an **Executor**, receiving an **Outcome**
with full resource accounting, scoring it with an **Evaluator**, and writing the whole record to a
**Ledger**. Later phases let the kernel make more of the decisions inside that plan. The loop does
not change.

Measurement is not a separate harness bolted onto the side. The ledger and the evaluator are how the
kernel decides, not only how it is graded.

---

## What Ration is not

| Not | Because |
|---|---|
| A model | Ration uses existing open weight models as controlled experimental subjects. It trains nothing in phases 1 to 5 |
| An inference engine | vLLM, Ollama, llama.cpp and MLX already do this well. Ration decides what to send them |
| A serving layer | Queuing, admission control, fairness and retries belong to a serving layer. See [relationship to other projects](#relationship-to-other-projects) |
| A RAG library | Retrieval is one way to satisfy an information need. Ration decides whether an information need exists, how much of it to satisfy, and at what resolution |
| An AGI claim | This is an engineering and measurement problem. Nothing here is evidence about general intelligence |

---

## The kernel loop

```
Task
  -> Appraiser      -> TaskProfile   difficulty, information need, budget class
  -> Planner        -> Plan          context set at resolutions, model, budgets, strategy
  -> Executor port  -> Outcome       answer plus full resource accounting
  -> Evaluator      -> Verdict       quality, sufficiency signal, failure mode
  -> Ledger         -> record        persisted, hardware labelled, reproducible
  -> Policy store   -> policy        arrives in phase 6
```

Each phase adds a decision the kernel is allowed to make. It adds nothing else. Phase 1 lets the
kernel choose the context set under a token budget. Phase 2 lets it choose resolution and tier.
Phase 4 lets it choose the model and the reasoning depth. The loop above is the same in every phase.

---

## Ports

Everything the kernel talks to is a port with more than one implementation, so the kernel can be
tested without a model, a GPU or a network.

| Port | Responsibility |
|---|---|
| `Executor` | Run a `Plan`, return an `Outcome` with resource accounting |
| `InformationStore` | Hold items at multiple resolutions and return item `X` at resolution `R` |
| `Selector` | Choose a subset of available information under a budget |
| `Evaluator` | Score answer quality and detect insufficiency |
| `Meter` | Measure the resources one run consumed |
| `Ledger` | Persist plan, outcome, verdict and hardware profile |

---

## The contracts

`Plan` and `Outcome` are the only things that cross the boundary between the kernel and whatever
executes the work. They are versioned data, not code paths, so any plan is replayable, diffable and
testable against any other plan.

**Plan**, by the phase that introduces each field:

| Field | Phase |
|---|---|
| `task_id`, `model`, `token_budget`, `latency_budget`, `cost_budget` | 1 |
| `context_items[]` with `item_id` and `resolution` | 1 |
| `page_set`, `tier_placement`, `prefetch_hints` | 2 |
| `expansion_policy` | 3 |
| `reasoning_depth`, `strategy`, `fallback_ladder` | 4 |
| `memory_scope`, `as_of` | 5 |
| `policy_version` | 6 |

**Outcome**:

| Field | Meaning |
|---|---|
| `tokens_in`, `tokens_out`, `context_tokens_materialised` | What was actually paid for |
| `wall_ms`, `ttft_ms` | Latency |
| `memory_peak_bytes`, `kv_bytes`, `bytes_moved` | Memory and transfer |
| `energy_j`, `cost_usd` | Where measurable. Null where not |
| `failure_mode` | Null, or one value from a closed set |

Null is a first class value. Ollama on Apple silicon cannot report a KV cache size, so the record
says null rather than a plausible estimate. That rule is a large part of what separates this from a
system that wins on its own benchmark.

---

## Honesty rules

These are enforced in code wherever enforcement is possible, not just stated here.

| Rule | How it is enforced |
|---|---|
| No number appears in this repository unless it came from a run recorded in the ledger | The report generator reads the ledger. There is no path that writes a number by hand |
| Every claim carries hardware profile, model id and quantisation, seed, date, and the baseline it beat | Required fields on the record. Reports refuse to render without them |
| Results from different hardware profiles are never compared | The ledger refuses the comparison and says why |
| Baselines are code in this repository, not descriptions in prose | Full context, fixed truncation, plain retrieval, and oracle context as an upper bound |
| A direction is not called novel until the closest existing systems are named and the gap is stated | Every direction has a landscape entry naming prior art |

---

## How quality per resource is reported

Not as one number. Collapsing quality and cost into a single scalar hides the exact tradeoff the
project exists to study, and it lets any system look good by choosing a weighting.

Ration reports a **frontier**: quality on one axis, a chosen resource on the other, with every
baseline drawn on the same axes from the same ledger.

The headline result is the **oracle gap**: how far the selector is from the best possible selection
for that task, measured against an oracle that is allowed to see the answer. A small oracle gap at a
small materialised fraction is the claim worth making. "Supports ten million tokens" is not a claim,
it is a configuration.

---

## Phases

Each phase ships a component in its final shape. No phase ships a version that a later phase throws
away. Detail and exit criteria live in [ROADMAP.md](ROADMAP.md).

| Phase | Name | The decision the kernel gains |
|---|---|---|
| 1 | Kernel and ledger | What is the minimum sufficient context for this task under a token budget |
| 2 | Virtual context | Which page, at which resolution, in which tier, and when to fault, promote, evict and prefetch |
| 3 | Reversible context | When compressed is not enough, and what to expand |
| 4 | Compute policy | Which model, how much reasoning, under a joint budget |
| 5 | Memory compiler and evolution | What experience becomes, and how it ages, contradicts and consolidates |
| 6 | Learning and self evaluation | Which plans work for which task shapes, learned from the ledger |
| 7 | Runtime | Every decision under one joint budget optimiser |

---

## Hardware

The primary development and measurement machine is Apple silicon with unified memory. Other
hardware is used when it is available at no cost.

This has a consequence that is stated plainly rather than hidden: on unified memory there is no
separate GPU memory, CPU memory and host transfer path to move data between, so the tier placement
half of phase 2 cannot be demonstrated on the primary machine. The information layer of phase 2 can.
Every record carries its hardware profile, and results from different profiles are never compared.

---

## Relationship to other projects

| Project | Boundary |
|---|---|
| [contextos](https://github.com/ranjan-del/contextos) | The long horizon research agenda that Ration serves. Ration is one implementation programme against it |
| [loomrun](https://github.com/ranjan-del/loomrun) | Serving layer for one machine: admission, queuing, priority, placement, retries. Loomrun answers how to run a request safely and fairly. Ration answers what request should exist. Loomrun is one `Executor` adapter among several, and Ration never requires it |
| [ragfabric](https://github.com/ranjan-del/ragfabric) | A retrieval platform. Retrieval is one way to satisfy an information need. Ration can drive it through the `InformationStore` port, and does not depend on it |

---

## Prior art

Ration is built in a crowded field and says so. The closest existing work for each direction, what
it does well, and the specific gap that remains, is written up in
[docs/landscape.md](docs/landscape.md). A short version:

| Area | Closest existing work |
|---|---|
| Hierarchical memory and virtual context | MemGPT and Letta, LMCache, Mooncake, NVMe cache offload tiers |
| Query aware page selection and eviction | Quest, H2O, RocketKV, and the 2026 adaptive eviction literature |
| Routing and cascades | RouteLLM, FrugalGPT, calibrated uncertainty cascade routing |
| Agent memory and lifecycle | Mem0, Zep and Graphiti, A-MEM, MemoryLACE, bitemporal contradiction resolution |
| Compression with recovery | Compress gather recompute, and cache blending |
| Evaluation | RULER, HELMET, BABILong, LongMemEval, LoCoMo, BEAM |

Ration does not claim to beat these systems at the thing each of them is best at. The claim it
intends to test is narrower and, if it holds, more useful: that one policy deciding tier,
resolution, model and budget jointly, under one metric, beats the best combination of point
solutions tuned separately.

---

## Repository layout

```
packages/
  ration-core/      kernel, contracts, ports. No I/O
  ration-exec/      executor adapters
  ration-eval/      evaluator, baselines, task suites, metrics
  ration-ledger/    persistence, hardware profiles, reporting
  ration-cli/       ration run, ration compare, ration report
docs/
  adr/              architecture decision records
  concepts/         explanations of the ideas the system rests on
  design/           design specifications
  plans/            phase implementation plans
  benchmarks/       measured results, one file per phase
  problem.md        the problem statement and non goals
  landscape.md      prior art, per direction, with the gap stated
  architecture.md   the kernel, the ports, the contracts
  evaluation.md     how anything here is allowed to be measured
```

The `packages/` tree is described by the phase 1 plan and does not exist yet.

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). The honesty rules above apply to contributions as strictly
as they apply to the maintainer.

---

## License

Apache License 2.0. See [LICENSE](LICENSE) and [NOTICE](NOTICE).
