# Landscape

Ration is built in a crowded field. This document names the closest existing work for each research
direction, states what those systems do well, and states the specific measurable gap that remains.

A direction is not called novel in this repository until it has an entry here.

Last reviewed: 2026-09-22. Sources are listed at the end. Where a claim comes from a secondary source
rather than the paper itself, it is marked as such and should be checked against the primary source
before it is relied on in a published result.

---

## 1. Hierarchical memory and virtual context

**Closest existing work.** MemGPT introduced virtual context management by analogy with operating
system virtual memory, giving an agent a small in context core and paging the rest in and out through
function calls. It became Letta, which organises memory into a core block that always stays resident
and recall memory that is searched on demand, and which later added asynchronous consolidation
between turns. On the serving side, LMCache extracts KV caches out of GPU memory and reuses them
across queries and engines, with backends spanning CPU memory, local SSD, Redis, object storage and
Mooncake. Mooncake aggregates memory across nodes into a distributed KV pool and is integrated with
vLLM. Practitioner material describes a three tier arrangement of GPU memory, host memory and NVMe
with roughly two orders of magnitude of bandwidth between the ends.

**What they do well.** Letta solved persistence and gave the agent agency over its own memory.
LMCache and Mooncake solved KV reuse and movement at production scale and speed. None of this should
be rebuilt.

**The gap.** These are two separate layers that do not talk to each other. Letta decides what
information the agent should see, with no knowledge of what any of it costs to materialise. LMCache
and Mooncake decide where bytes live, with no knowledge of what the task needs. Nothing decides both
under one budget, and nothing measures the pair together. That joint decision is what Ration exists
to test.

---

## 2. Demand paging and page selection

**Closest existing work.** Quest performs query aware KV page selection, estimating page criticality
from lightweight metadata and loading only the top ranked pages. H2O retains heavy hitter tokens by
accumulated attention, balanced against recency. RocketKV splits a target compression ratio across
two stages. The 2026 literature adds learned eviction with latent memory, output perturbation and
attention output error criteria for choosing what to drop, value aware stochastic eviction for
reasoning models, predictive online pruning, adaptive per head budgets and uncertainty gated block
sparse selection. There is also work on cooperative memory paging for long horizon conversation using
keyword bookmarks.

**What they do well.** Selection at the KV and attention level, with real speedups, and increasingly
principled criteria for what to keep.

**The gap.** Two of them. First, selection happens at one fixed resolution: a page is either resident
or evicted. There is no notion of the same information existing as raw tokens, as a summary, and as a
set of facts, with the runtime choosing which form to pay for. Second, every one of these methods
reports against the baselines its authors chose, and the field has no neutral harness. The 2026
eviction literature is repeating the pattern that unified routing evaluation has already exposed.

---

## 3. Routing, cascades and compute budgets

**Closest existing work.** FrugalGPT introduced cascade routing with large reported cost reductions.
RouteLLM routes between a weak and a strong model and reports large cost reductions at most of the
strong model's quality. The 2026 work adds calibrated uncertainty for cost optimal cascade routing,
and a survey framing the design space by when the decision is made, what information feeds it and how
it is computed.

**What they do well.** Establishing that most queries do not need the strongest model, and giving
practical mechanisms for acting on that.

**The gap, and it is unusually clear.** Unified evaluation work published in 2026 reports that router
performance is highly benchmark dependent, that under fair unified evaluation most routing methods
collapse to similar performance, and that several recent approaches fail to reliably beat simple
baselines. Practitioner reports of production savings span a wide range and depend entirely on the
traffic and the model pair.

This is the single most important finding in this document. It says the field's problem is not a
shortage of methods, it is a shortage of trustworthy measurement. Ration's phase 4 therefore has a
publishable negative result available to it, and the roadmap says so explicitly.

Note also that routing decisions in this literature are made in isolation from context decisions. A
smaller model with better selected context may beat a larger model with worse selected context, and
no current system is allowed to make that trade because no current system owns both decisions.

---

## 4. Memory compilation

**Closest existing work.** Mem0 extracts salient information from conversation, reconciles it with
what is stored and retrieves memory objects across sessions. A-MEM, SimpleMem, AtomMem, MemoryBank
and All-Mem study extraction, update, consolidation, retrieval and revision. SimpleMem emphasises
semantic compression, recursive consolidation and query aware retrieval. Graph based approaches
including HippoRAG and Graphiti compile experience into entities and relationships.

**What they do well.** Turning conversation into something more useful than conversation, and showing
that it measurably helps on memory benchmarks.

**The gap.** Compilation is treated as lossy summarisation with retrieval on top. The original is
usually not recoverable, and where it is, nothing decides when recovery is worth its cost. Ration's
phase 3 and phase 5 are specifically about the recoverable case: compress hard, detect insufficiency,
expand the specific region that is insufficient, and measure what the round trip cost.

---

## 5. Memory lifecycle and evolution

**Closest existing work.** This is the most active area of 2026 in this list. A four lever framing of
consolidation covers importance, merge, decay and eviction, with Mem0, Zep, Letta, LangChain and
others placing those levers differently. MemoryLACE addresses lifecycle aware consolidation and
evidence retrieval. TOKI proposes a bitemporal operator algebra for contradiction resolution in
persistent agent memory. Other 2026 work studies retention consequences in lifecycle control and how
control plane placement shapes forgetting across system configurations. Memora introduces a metric
for forgetting aware memory accuracy.

**What they do well.** Taking seriously that knowledge changes, that contradiction is normal and that
forgetting is a feature. Bitemporal reasoning in particular is the right shape for the superseded
preference problem.

**The gap.** Lifecycle is decided independently of cost. A memory is consolidated or decayed on
importance and recency, not on what its various representations cost to keep and to use. Ration's
contribution here, if any, is to put lifecycle transitions under the same budget as everything else.

**Honest assessment.** This is the most crowded direction in the entire programme and the one where
Ration is least likely to be novel. It is scheduled late for that reason.

---

## 6. Compression with recovery

**Closest existing work.** Compress gather recompute style methods reform long context processing by
compressing and then recomputing what is needed. Cache blending composes cached pieces. Adaptive KV
cache reuse targets fast long context serving.

**What they do well.** Showing that the compress and recompute round trip is viable.

**The gap.** The trigger. Existing methods recompute on a schedule or on a fixed policy. Nothing asks
the model whether what it currently has is sufficient and expands only the region it says is not. The
insufficiency signal is the interesting part and it is underexplored.

---

## 7. Prefetch and predicted information demand

**Closest existing work.** Prefetching is standard in the serving literature, and sleep time compute
in Letta performs work between turns. Query aware selection methods implicitly predict one step
ahead.

**The gap.** Prediction is single step and reactive. Nothing uses task state, reasoning state, memory
access history and workflow state together to predict demand several steps out and overlap movement
with computation. This is genuinely open, and it is also the hardest to demonstrate on hardware
without separate memory tiers.

---

## 8. Continual learning and self improvement

**Closest existing work.** The continual learning, test time adaptation, lightweight adapter, skill
library, experience replay and model editing literatures. On the verification side, self critique and
retrieval augmented self assessment.

**The gap.** For Ration the interesting object is not the model, it is the policy. The ledger
accumulates thousands of records of the form: this task shape, this plan, this cost, this quality.
Learning a better planner from that is a supervised problem on data the system already produces, and
it does not require touching model weights. That framing is the phase 6 bet.

**Honest assessment.** Capital intensive if approached as model adaptation. Cheap if approached as
policy learning over the ledger. Ration takes the second path, and should say so rather than imply
it is doing the first.

---

## 9. Evaluation and benchmarks

**Closest existing work.** RULER for synthetic long context tasks, HELMET for real tasks across multi
hop question answering, summarisation and code, InfiniteBench and LongBench for long context, and
BABILong for reasoning over facts distributed through long text. For memory, LongMemEval, LoCoMo and
BEAM, the last extending to ten million tokens. Reported leaderboard positions on LongMemEval and
BEAM exist for several commercial memory systems.

**What they do well.** BABILong in particular produces the finding that anchors this project: models
effectively use only a small fraction of the context they nominally support. By 2026 the field
broadly treats long context and memory as different problems with different evaluations.

**The gap.** All of these measure quality. None of them measures quality against resources consumed.
A system can top LongMemEval by materialising everything, and the benchmark will not notice. There is
no public harness that draws quality and cost on the same axes for a set of systems on the same
hardware. That harness is Ration's phase 1, and it is the part most likely to be useful to other
people regardless of whether the rest of the programme succeeds.

---

## Summary of where the open space is

| Direction | How crowded | Ration's position |
|---|---|---|
| 1. Hierarchical memory | Very crowded, production grade | Do not rebuild. Build the joint decision the layers do not make |
| 2. Page selection | Very crowded, active | Add multi resolution, and measure neutrally |
| 3. Routing | Crowded and partly discredited | Test honestly, publish a negative result if that is what the data says |
| 4. Memory compilation | Crowded | Focus on recoverability and the cost of recovery |
| 5. Memory lifecycle | Most crowded of all | Least likely to be novel. Scheduled late |
| 6. Compression with recovery | Partly open | The insufficiency trigger is the open part |
| 7. Predicted demand | Open | Hardest to demonstrate on available hardware |
| 8. Continual learning | Crowded as model adaptation, open as policy learning | Policy learning over the ledger |
| 9. Evaluation of cost and quality together | Open | Phase 1. The most likely thing here to be useful to others |

---

## Sources

Reviewed 2026-09-22 through literature and documentation search. Primary papers should be read before
any of this is cited in a published result.

- MemGPT: Towards LLMs as Operating Systems, https://arxiv.org/abs/2310.08560
- LMCache, https://github.com/lmcache/lmcache and https://docs.lmcache.ai/kv_cache/mooncake.html
- Mooncake, https://kvcache-ai.github.io/Mooncake/
- From Tensor Buffer to Distributed Memory Hierarchy: A Survey of KV Cache Management for LLM Serving, https://arxiv.org/pdf/2607.02574
- Adaptive KV Cache Reuse for Fast Long-Context LLM Serving, https://arxiv.org/pdf/2605.24022
- NVMe KV cache offloading tiers, https://www.spheron.network/blog/nvme-kv-cache-offloading-llm-inference/ (secondary)
- IndexMem: Learned KV-Cache Eviction with Latent Memory, https://arxiv.org/pdf/2605.25475
- CriticalKV, https://arxiv.org/pdf/2502.03805
- CAOTE: KV Cache Selection via Attention Output Error-Based Token Eviction, https://arxiv.org/html/2504.14051v6
- RocketKV, https://arxiv.org/pdf/2502.14051
- AhaKV, https://arxiv.org/pdf/2506.03762
- Uncertainty-gated selection for block-sparse attention, https://arxiv.org/pdf/2607.07724
- Cooperative Memory Paging with Keyword Bookmarks for Long-Horizon LLM Conversations, https://arxiv.org/pdf/2604.12376
- UCCI: Calibrated Uncertainty for Cost-Optimal LLM Cascade Routing, https://arxiv.org/html/2605.18796
- Memory for Autonomous LLM Agents: Mechanisms, Evaluation, and Emerging Frontiers, https://arxiv.org/pdf/2603.07670
- MemoryLACE: Memory Lifecycle-Aware Consolidation and Evidence Retrieval, https://arxiv.org/html/2609.03201
- TOKI: A Bitemporal Operator Algebra for Contradiction Resolution in LLM-Agent Persistent Memory, https://arxiv.org/pdf/2606.06240
- Retention Consequence in Lifecycle Memory Control, https://arxiv.org/pdf/2604.16774
- Control-Plane Placement Shapes Forgetting, https://arxiv.org/pdf/2606.15903
- Agent Memory Paper List, https://github.com/Shichun-Liu/Agent-Memory-Paper-List
- Compress, Gather, and Recompute: REFORMing Long-Context Processing in Transformers, https://arxiv.org/pdf/2506.01215
- The consolidation problem in agent memory, https://hindsight.vectorize.io/blog/2026/05/21/agent-memory-consolidation (secondary)
- AI memory benchmarks 2026, https://mem0.ai/blog/ai-memory-benchmarks-in-2026 (secondary)
- LLM model routing in 2026, https://www.digitalapplied.com/blog/llm-model-routing-2026-cost-quality-optimization-engineering-guide (secondary)
