# Changelog

All notable changes to this project are documented here.
The format follows Keep a Changelog, and this project adheres to Semantic Versioning.

## [Unreleased]

### Added

- Problem statement, non goals, and the conditions that would falsify the thesis
- Landscape review of prior art across nine research directions, with the specific measurable gap
  stated for each and sources listed
- Architecture: the kernel loop, six ports, and the Plan, Outcome, Verdict, TaskProfile and
  HardwareProfile contracts
- Evaluation method: honesty rules, four baselines, the quality against resource frontier, the oracle
  gap as the headline result, and reproducibility requirements
- Design specification recording the approaches considered and why the policy kernel was chosen
- ADR 0001: Ration is a policy kernel over pluggable executors
- ADR 0002: the seam between Ration and any serving layer is the Plan and Outcome contract
- ADR 0003: measurement lives inside the kernel, not in a separate harness
- ADR 0004: every record carries a hardware profile, unmeasurable values are null, and profiles are
  never compared
- ADR 0005: Plan is versioned data, and phases add fields rather than code paths
- Roadmap with seven phases and an exit criterion for each
- Project configuration: Python 3.13, uv workspace, ruff, pytest

### Notes

- No implementation code yet. The packages tree is described by the phase 1 plan and does not exist
- No measured numbers appear anywhere in this repository, because no run has been recorded
