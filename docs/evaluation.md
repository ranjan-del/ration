# Evaluation

How anything in this repository is allowed to be measured, and what is allowed to be claimed.

## The rules

| Rule | Enforcement |
|---|---|
| No number appears in this repository unless it came from a run recorded in the ledger | The report generator reads the ledger. There is no code path that writes a number by hand, and a number in prose that the ledger cannot reproduce is a bug |
| Every claim carries hardware profile, model id and quantisation, seed, date and the baseline it beat | Required fields on the record. Report rendering fails if any are missing |
| Results from different hardware profiles are never compared | The ledger refuses the comparison and states the two profiles |
| Baselines are code, not prose | They live in `ration-eval` and run in the same harness as the system under test |
| A direction is not called novel until the closest existing systems are named and the gap is stated | Every direction has an entry in `landscape.md` |
| Negative results are published | A phase whose exit criterion is not met reports what happened and why, in `docs/benchmarks/` |

## The baselines

Four, all first class code, all run on the same harness as the system under test.

| Baseline | What it does | Why it is here |
|---|---|---|
| Full context | Materialise everything available, up to the model limit | The thing everyone actually does. If Ration does not beat this, there is no project |
| Fixed truncation | Take the first or last N tokens | Embarrassingly simple, and it beats sophisticated methods more often than the literature admits |
| Plain retrieval | Standard top-k retrieval over the same information | The other thing everyone actually does |
| Oracle context | Select using knowledge of the answer | An upper bound. Nothing can beat it. The distance to it is the honest headline |

## The metric

Not a single scalar. Quality and cost are reported as a frontier: quality on one axis, a chosen
resource on the other, every baseline drawn on the same axes from the same ledger.

Collapsing quality and cost into one number hides the tradeoff this project exists to study, and lets
any system look good by choosing the weighting after seeing the results.

### Resources measured

| Resource | Unit | Available on Apple silicon |
|---|---|---|
| Context tokens materialised | tokens | Yes |
| Generated tokens | tokens | Yes |
| Wall time | ms | Yes |
| Time to first token | ms | Yes |
| Peak memory | bytes | Yes, unified |
| KV cache size | bytes | No, recorded as null |
| Bytes moved between tiers | bytes | Not applicable, recorded as null |
| Energy | joules | Partially, platform dependent |
| Money | USD | Only for metered remote executors |

Null means not measurable on this profile. It never means zero, and it is never replaced with an
estimate.

## The headline: oracle gap

The claim worth making is a small oracle gap at a small materialised fraction:

> On task family F, at hardware profile P, with model M at quantisation Q, Ration reached X percent
> of oracle quality while materialising Y percent of available context, against full context at Z
> percent of oracle quality at 100 percent materialised. Seed S, date D.

Every part of that sentence comes from the ledger.

## Task suites

Chosen for what each one can falsify, not for what it is likely to show. Selection and the reason for
each is phase 0 work and is recorded here when complete.

Candidate families under consideration, with what each tests:

| Family | Tests |
|---|---|
| Synthetic retrieval over long inputs | Whether selection finds a known target. Cheap, controllable, and weak evidence on its own |
| Multi hop question answering over documents | Whether selection survives needing several pieces at once |
| Long horizon conversation | Whether memory across sessions holds, including contradiction and time |
| Code understanding over a repository | A real task family where the available information vastly exceeds any context window |

## Reproducibility

Every recorded run captures the seed, the model id and quantisation, the executor adapter and its
version, the hardware profile, the package versions and the task suite version. A run that cannot be
reproduced from its own record is a defect in the ledger, not an acceptable result.
