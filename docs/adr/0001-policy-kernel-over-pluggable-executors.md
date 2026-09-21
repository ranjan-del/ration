# ADR 0001: Ration is a policy kernel over pluggable executors

- Status: accepted
- Date: 2026-09-22

## Context

The research agenda in `contextos` describes a runtime that manages context, memory, KV cache, model
selection, reasoning budget, scheduling, retrieval, compression, continual learning and evaluation.
Taken literally, that is an inference engine, a serving layer, a retrieval platform, a memory system
and an evaluation harness in one repository.

Three of those already exist as strong open source systems, described in `landscape.md`. Rebuilding
them would consume the entire programme and produce something slower than what it replaced.

## Decision

Ration is a policy kernel. It decides what to materialise, at what resolution, on which model, under
what budget. It hands that decision to an executor as a `Plan` and receives an `Outcome`. It does not
run models, does not schedule requests and does not own storage.

Everything the kernel touches is a port with at least two implementations, one of which is
deterministic and offline.

## Consequences

Positive:

- The programme can use vLLM, Ollama, MLX and llama.cpp rather than competing with them.
- The kernel is testable with no model, no accelerator and no network.
- Baselines and Ration policies are interchangeable implementations of the same port, which makes
  comparison structurally fair rather than fair by good intentions.
- Hardware portability is a property of the adapter layer, not of the research.

Negative:

- Resource accounting is limited by what each executor exposes. Ollama cannot report a KV cache size,
  so that field is null on the primary hardware. ADR 0004 covers how that stays honest.
- Some optimisations that need engine internals are out of reach without writing an adapter that
  reaches into vLLM. That is accepted, and deferred until a CUDA machine exists.

## Alternatives rejected

| Alternative | Why rejected |
|---|---|
| Build the runtime as a fork of an inference engine | Couples the research to one engine and one hardware family, and puts the interesting work behind a large amount of uninteresting work |
| Build as a library called from application code with no kernel loop | No place for the ledger, so no measurement, so no way to tell whether any of it works |
