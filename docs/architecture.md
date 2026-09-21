# Architecture

## The kernel loop

One loop. Every phase adds a decision the kernel is allowed to make inside it. The loop itself does
not change.

```
Task
  -> Appraiser      -> TaskProfile    difficulty, information need, budget class
  -> Planner        -> Plan           what to materialise, on what, under what budget
  -> Executor port  -> Outcome        the answer plus full resource accounting
  -> Evaluator      -> Verdict        quality, sufficiency signal, failure mode
  -> Ledger         -> Record         persisted, hardware labelled, reproducible
  -> Policy store   -> Policy         phase 6 onwards
```

Read the phases as a table of what the planner is permitted to decide:

| Phase | Planner may decide |
|---|---|
| 1 | Which information items, under a token budget |
| 2 | Also: at what resolution, in which tier, and when to fault, promote, demote, evict, prefetch |
| 3 | Also: when to expand something compressed, and which region |
| 4 | Also: which model, how much reasoning, and what to give up first when the budget binds |
| 5 | Also: which compiled memory, as of when, in which lifecycle state |
| 6 | All of the above, from a policy learned on the ledger |
| 7 | All of the above, jointly optimised rather than decided in sequence |

## Ports

`ration-core` defines ports and contains no I/O. Everything that touches a model, a disk, a clock or
a network sits behind a port with at least two implementations, one of which is deterministic and
offline.

| Port | Responsibility | Why it is a port |
|---|---|---|
| `Executor` | Run a `Plan`, return an `Outcome` | So the kernel works on Ollama, MLX, vLLM, a remote API or a deterministic double without changing |
| `InformationStore` | Return item `X` at resolution `R`, and describe what exists | So the resolution ladder can change without touching the planner |
| `Selector` | Choose a subset under a budget | So baselines and Ration policies are interchangeable and directly comparable |
| `Evaluator` | Score quality, detect insufficiency | So graders can be swapped per task family, and an offline grader keeps tests hermetic |
| `Meter` | Measure one run's resource use | So the platform specific parts stay in one place and unmeasurable values stay null |
| `Ledger` | Persist and query records | So SQLite locally and Postgres for larger studies are the same interface |

The `Selector` port is worth a note. Baselines are implementations of it. Full context, fixed
truncation, plain retrieval and the oracle are all selectors. That is what makes comparison fair: the
system under test and the thing it claims to beat differ in exactly one object.

## The contracts

`Plan` and `Outcome` are the only things that cross the executor boundary. They are versioned data.

Three properties are load bearing:

| Property | Consequence |
|---|---|
| A `Plan` is data, not a code path | Any plan can be serialised, replayed, diffed against another plan, and executed by any executor |
| A phase adds fields to `Plan`, never branches to the kernel | The kernel does not grow a conditional per phase, which is the usual way a system like this rots |
| An `Outcome` may contain null | Null means not measurable on this hardware profile. It never means zero, and it is never estimated |

### Plan

| Field | Phase | Meaning |
|---|---|---|
| `plan_version` | 1 | Contract version |
| `task_id` | 1 | The task this plan answers |
| `model` | 1 | Model id and quantisation |
| `token_budget`, `latency_budget`, `cost_budget` | 1 | Limits the executor must respect |
| `context_items[]` | 1 | Item id plus resolution |
| `page_set`, `tier_placement`, `prefetch_hints` | 2 | Paging decisions |
| `expansion_policy` | 3 | When and what to expand |
| `reasoning_depth`, `strategy`, `fallback_ladder` | 4 | Compute decisions |
| `memory_scope`, `as_of` | 5 | Which memory, at what point in time |
| `policy_version` | 6 | Which learned policy produced this plan |

### Outcome

| Field | Meaning |
|---|---|
| `answer` | The produced answer |
| `tokens_in`, `tokens_out`, `context_tokens_materialised` | What was paid for |
| `wall_ms`, `ttft_ms` | Latency |
| `memory_peak_bytes`, `kv_bytes`, `bytes_moved` | Memory and transfer, null where not measurable |
| `energy_j`, `cost_usd` | Null where not measurable |
| `failure_mode` | Null, or one value from a closed set |
| `executor_id`, `executor_version` | Which implementation ran it |

### Verdict

| Field | Meaning |
|---|---|
| `quality` | Score from the grader for this task family |
| `sufficiency` | Whether the model signalled it had enough information. Drives phase 3 |
| `failure_diagnosis` | Where the loop went wrong, when it did |

### HardwareProfile

Attached to every record. Two records with different profiles cannot be compared, and the ledger
enforces that rather than trusting the reader to notice.

| Field | Example |
|---|---|
| `profile_id` | Stable identifier for this machine and configuration |
| `platform` | Apple silicon unified memory, or CUDA discrete |
| `memory_bytes` | Total, and the split where one exists |
| `accelerator` | Model and count |
| `measurable[]` | Which `Outcome` fields this profile can actually fill |

## Where state lives

| State | Home | Lifetime |
|---|---|---|
| Information items at all resolutions | `InformationStore` | Long lived, rebuildable from source |
| Run records | `Ledger` | Permanent. The repository's only source of numbers |
| Learned policy | Policy store | Versioned, and every plan records which version made it |
| Task suites | Repository | Versioned with the code, because a suite change invalidates comparisons |
| Nothing else | | The kernel is stateless between runs |

## Packages

| Package | Contains | Depends on |
|---|---|---|
| `ration-core` | Contracts, ports, the kernel loop. No I/O | Nothing |
| `ration-exec` | Executor adapters and meters | `ration-core` |
| `ration-eval` | Evaluators, baselines, task suites, metrics | `ration-core` |
| `ration-ledger` | Persistence, hardware profiles, report generation | `ration-core` |
| `ration-cli` | `ration run`, `ration compare`, `ration report` | All of the above |

Import direction is enforced in CI, as it is in RagFabric. `ration-core` importing anything from the
other packages is a build failure, not a review comment.

## Offline first

Every test passes with no API key and no network, through deterministic doubles behind the `Executor`
and `Evaluator` ports. Real models sit behind configuration as an optional path. This is the same
decision taken across the rest of the portfolio, and it is why continuous integration stays green
without secrets and the whole system stays demonstrable on a laptop.
