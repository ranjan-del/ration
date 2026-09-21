# Documentation

| Document | What it covers |
|---|---|
| [problem.md](problem.md) | The question, the four observable problems, the non goals, and what would falsify the thesis |
| [landscape.md](landscape.md) | Prior art per direction, what each existing system does well, and the specific measurable gap |
| [architecture.md](architecture.md) | The kernel loop, the ports, the contracts, and where state lives |
| [evaluation.md](evaluation.md) | The honesty rules, the baselines, the metric, the task suites, reproducibility |
| [design/](design/) | Design specifications |
| [plans/](plans/) | Phase implementation plans |
| [adr/](adr/) | Architecture decision records |
| [concepts/](concepts/) | Explanations of the ideas the system rests on |
| [benchmarks/](benchmarks/) | Measured results, one file per phase, generated from the ledger |

## Architecture decision records

| ADR | Decision |
|---|---|
| [0001](adr/0001-policy-kernel-over-pluggable-executors.md) | Ration is a policy kernel over pluggable executors, not an inference engine or a serving layer |
| [0002](adr/0002-serving-layer-seam.md) | The seam between Ration and any serving layer is the Plan and Outcome contract |
| [0003](adr/0003-measurement-inside-the-kernel.md) | Measurement lives inside the kernel, not in a separate harness |
| [0004](adr/0004-hardware-profiles-and-null.md) | Every record carries a hardware profile, unmeasurable values are null, and profiles are never compared |
| [0005](adr/0005-plan-is-data.md) | Plan is versioned data. Phases add fields, not code paths |
