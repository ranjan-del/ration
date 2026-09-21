# The problem

## The question

Can AI systems become more capable by managing information, memory, computation and reasoning far
better, rather than by continuously increasing model size and context length?

## The objective

One ratio:

**useful answer quality per unit of context, computation, memory, latency, energy and money.**

Both halves matter. A system that halves cost and halves quality has achieved nothing. A system that
holds quality while materialising a small fraction of the available information has achieved the
thing this project exists to find.

## Four observable problems

### 1. Nominal context is not effective context

Models advertise context windows measured in hundreds of thousands or millions of tokens. Published
long context evaluation finds that models effectively use a small fraction of what they nominally
support. Filling a window is not the same as using it, and paying for a filled window is not the same
as buying quality.

The consequence for this project: a claim of the form "supports N tokens" is not a result. It is a
configuration. The result is what quality survives at what materialised fraction.

### 2. Every decision is made at maximum

Production systems default to the largest model, the full retrieved set and the maximum reasoning
budget, for every request, regardless of whether the task needed any of it. The cost is paid whether
or not it bought anything, and nobody finds out, because the counterfactual is never run.

### 3. Point solutions are benchmarked in isolation

Cache eviction, retrieval, routing, compression and agent memory are each strong, active fields. Each
system is measured on the benchmark its authors chose, against the baselines its authors chose.
Nothing measures them together, under one budget, against one metric. So a practitioner assembling
five of them has no way to know whether the combination helps.

### 4. Methods collapse under fair evaluation

Unified evaluations of model routing published in 2026 report that most methods perform similarly
once the evaluation is held constant, and that several fail to beat simple baselines. The same
pattern is beginning to appear in cache eviction and in agent memory.

This is the finding that shapes the whole project. The scarce thing is not another method. The scarce
thing is a neutral, reproducible way to tell whether a method works. So the measurement substrate is
built first, and it is built into the kernel rather than bolted on beside it.

## Non goals

| Non goal | Why |
|---|---|
| Training a foundation model | Existing open weight models are used as controlled experimental subjects. Phases 1 to 5 train nothing |
| Building an inference engine | vLLM, Ollama, llama.cpp and MLX do this well. Ration decides what to send them |
| Building a serving layer | Admission, queuing, fairness, retries and placement belong elsewhere. See ADR 0002 |
| Beating any one point solution at its own speciality | The claim under test is about combination and joint budgeting, not about winning a single benchmark |
| Claiming anything about general intelligence | This is an engineering and measurement problem |

## What would falsify the thesis

Stated in advance, so the project can be wrong rather than merely unfalsifiable.

| Thesis | What would falsify it |
|---|---|
| Minimum sufficient context beats full context on the quality per resource frontier | Full context wins at every budget on every task family, and the oracle gap is large enough that no selector could close it |
| One joint policy beats the best combination of separately tuned point solutions | Separately tuned point solutions match or beat the joint policy once both are measured on the same harness |
| Task difficulty can be appraised well enough to route on | Appraisal confidence is not calibrated, so routing on it does no better than a fixed cheap or fixed expensive policy. The 2026 routing literature already suggests this is a live risk |
| Compiled memory preserves task relevant information | Compiled representations lose exactly the information the tasks need, and expansion cannot recover it at acceptable cost |

A falsified thesis, measured properly and reported plainly, is a result this project will publish.
