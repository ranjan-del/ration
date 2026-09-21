# Ration Phase 1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship the Ration kernel with a ledger, a meter, an evaluator, four baselines and one real decision (choose the minimum sufficient context under a token budget), measured on code understanding over a real repository.

**Architecture:** A pure kernel in `ration-core` orchestrates appraise, plan, execute, evaluate, record. Everything that touches a model, a disk, a clock or a network sits behind a Protocol with at least two implementations, one of which is deterministic and offline. `Plan` and `Outcome` are versioned frozen dataclasses that reject unknown fields. The four baselines are implementations of the same `Selector` port as the Ration policy, so a comparison differs in exactly one object.

**Tech Stack:** Python 3.13, uv workspace, stdlib only in `ration-core`, sqlite3 for the ledger, pytest, ruff, import-linter. Ollama for real model runs. No network and no API key required for any test.

**Spec:** [`docs/design/2026-09-22-ration-design.md`](../design/2026-09-22-ration-design.md). Read it and [`docs/evaluation.md`](../evaluation.md) before starting. Supporting: [`docs/architecture.md`](../architecture.md), ADRs 0001 to 0005.

## Global Constraints

Every task's requirements implicitly include all of these.

- Python `>=3.13`. Workspace members live under `packages/*`.
- **`ration-core` has zero third party dependencies and imports nothing from the other packages.** Enforced by import-linter in CI. A violation is a build failure.
- **Offline first.** Every test passes with no API key and no network. Tests that need a model use the deterministic double.
- **No number is written by hand.** Anything numeric in `docs/benchmarks/` is produced by `ration report` reading the ledger.
- **Every record carries a `HardwareProfile`,** and the ledger refuses to compare records across profiles (ADR 0004).
- **Unmeasurable is `None`.** Never `0`, never an estimate. A derived value gets a differently named field.
- **`Plan` is versioned data.** Unknown fields are rejected, not ignored (ADR 0005).
- Conventional commits: `feat:`, `fix:`, `docs:`, `test:`, `refactor:`, `chore:`.
- **No em dashes** in code, comments, documentation or commit messages.
- No AI assistant authorship references in any commit message or trailer.
- Line length 100, ruff rules `E, F, I, N, UP, B, SIM, RUF`.

## File Structure

| Path | Responsibility |
|---|---|
| `packages/ration-core/src/ration_core/contracts.py` | `Plan`, `Outcome`, `Verdict`, `TaskProfile`, `HardwareProfile`, `ContextItem`, `Resolution`, `FailureMode`. Frozen dataclasses, versioned, strict decoding |
| `packages/ration-core/src/ration_core/ports.py` | `Executor`, `InformationStore`, `Selector`, `Evaluator`, `Meter`, `Ledger` as Protocols |
| `packages/ration-core/src/ration_core/kernel.py` | The loop. Pure orchestration, no I/O |
| `packages/ration-core/src/ration_core/errors.py` | `UnknownFieldError`, `ProfileMismatchError`, `BudgetExceededError` |
| `packages/ration-exec/src/ration_exec/double.py` | Deterministic offline executor |
| `packages/ration-exec/src/ration_exec/ollama.py` | Ollama executor adapter |
| `packages/ration-exec/src/ration_exec/meter_apple.py` | Apple silicon meter, and the honest nulls |
| `packages/ration-exec/src/ration_exec/profile.py` | Hardware profile detection |
| `packages/ration-ledger/src/ration_ledger/schema.py` | SQLite schema and migrations |
| `packages/ration-ledger/src/ration_ledger/store.py` | `SqliteLedger`, including the cross profile refusal |
| `packages/ration-ledger/src/ration_ledger/report.py` | Frontier and oracle gap rendering from the ledger only |
| `packages/ration-eval/src/ration_eval/corpus.py` | Repository to `InformationStore`, AST chunking |
| `packages/ration-eval/src/ration_eval/suite.py` | Task suite generation with exact ground truth |
| `packages/ration-eval/src/ration_eval/graders.py` | Exact and set match graders per question type |
| `packages/ration-eval/src/ration_eval/baselines.py` | `FullContext`, `FixedTruncation`, `PlainRetrieval`, `Oracle` |
| `packages/ration-eval/src/ration_eval/selector.py` | `BudgetedSelector`, the first Ration policy |
| `packages/ration-eval/src/ration_eval/metrics.py` | Frontier points and oracle gap |
| `packages/ration-cli/src/ration_cli/main.py` | `ration run`, `ration compare`, `ration report`, `ration suite build` |
| `tests/fixtures/minirepo/` | Tiny synthetic Python package used by every corpus and suite test |

---

### Task 1: Workspace, contracts and strict decoding

**Files:**
- Create: `packages/ration-core/pyproject.toml`
- Create: `packages/ration-core/src/ration_core/__init__.py`
- Create: `packages/ration-core/src/ration_core/errors.py`
- Create: `packages/ration-core/src/ration_core/contracts.py`
- Create: `packages/ration-core/tests/test_contracts.py`
- Modify: `pyproject.toml` (add `[dependency-groups]` with dev tools)

**Interfaces:**
- Consumes: nothing
- Produces: `Plan`, `Outcome`, `Verdict`, `TaskProfile`, `HardwareProfile`, `ContextItem`, `Resolution`, `FailureMode`, `UnknownFieldError`. Every contract class exposes `to_dict() -> dict` and classmethod `from_dict(d: dict) -> Self`. `PLAN_VERSION = 1`, `OUTCOME_VERSION = 1`.

- [ ] **Step 1: Write the failing test**

```python
# packages/ration-core/tests/test_contracts.py
import pytest

from ration_core.contracts import (
    PLAN_VERSION,
    ContextItem,
    HardwareProfile,
    Outcome,
    Plan,
    Resolution,
)
from ration_core.errors import UnknownFieldError


def _profile() -> HardwareProfile:
    return HardwareProfile(
        profile_id="test-profile",
        platform="test",
        memory_bytes=1,
        accelerator="none",
        measurable=("tokens_in", "tokens_out", "wall_ms"),
    )


def test_plan_round_trips_through_dict():
    plan = Plan(
        task_id="t1",
        model="double:v1",
        token_budget=100,
        context_items=(ContextItem(item_id="a", resolution=Resolution.RAW),),
    )
    assert Plan.from_dict(plan.to_dict()) == plan


def test_plan_carries_its_version():
    assert Plan(task_id="t1", model="m", token_budget=1).plan_version == PLAN_VERSION


def test_plan_rejects_unknown_fields_rather_than_ignoring_them():
    d = Plan(task_id="t1", model="m", token_budget=1).to_dict()
    d["tier_placement"] = "gpu"
    with pytest.raises(UnknownFieldError) as exc:
        Plan.from_dict(d)
    assert "tier_placement" in str(exc.value)


def test_plan_is_frozen():
    plan = Plan(task_id="t1", model="m", token_budget=1)
    with pytest.raises(AttributeError):
        plan.model = "other"


def test_outcome_keeps_unmeasurable_fields_as_none_not_zero():
    outcome = Outcome(
        answer="x",
        tokens_in=10,
        tokens_out=2,
        context_tokens_materialised=8,
        wall_ms=5.0,
        executor_id="double",
        executor_version="1",
    )
    assert outcome.kv_bytes is None
    assert outcome.energy_j is None
    assert outcome.cost_usd is None


def test_hardware_profile_reports_what_it_can_measure():
    assert _profile().can_measure("kv_bytes") is False
    assert _profile().can_measure("wall_ms") is True
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest packages/ration-core/tests/test_contracts.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'ration_core'`

- [ ] **Step 3: Write minimal implementation**

```python
# packages/ration-core/src/ration_core/errors.py
class RationError(Exception):
    """Base for every error this package raises."""


class UnknownFieldError(RationError):
    """A contract was decoded from a mapping containing fields it does not define."""


class ProfileMismatchError(RationError):
    """Two records from different hardware profiles were compared."""


class BudgetExceededError(RationError):
    """A plan asked for more than its stated budget allows."""
```

```python
# packages/ration-core/src/ration_core/contracts.py
"""Versioned data contracts. Stdlib only, by design. See ADR 0005."""

from __future__ import annotations

import dataclasses
from dataclasses import dataclass, field, fields
from enum import Enum
from typing import Any, Self

from ration_core.errors import UnknownFieldError

PLAN_VERSION = 1
OUTCOME_VERSION = 1


class Resolution(str, Enum):
    RAW = "raw"
    DETAILED = "detailed"
    SUMMARY = "summary"
    FACTS = "facts"
    SEMANTIC = "semantic"


class FailureMode(str, Enum):
    BUDGET_EXCEEDED = "budget_exceeded"
    EXECUTOR_ERROR = "executor_error"
    TIMEOUT = "timeout"
    EMPTY_ANSWER = "empty_answer"


class _Strict:
    """Decoding that rejects unknown fields instead of silently dropping them."""

    def to_dict(self) -> dict[str, Any]:
        return dataclasses.asdict(self, dict_factory=_encode)

    @classmethod
    def from_dict(cls, d: dict[str, Any]) -> Self:
        known = {f.name for f in fields(cls)}
        unknown = sorted(set(d) - known)
        if unknown:
            raise UnknownFieldError(
                f"{cls.__name__} does not define {', '.join(unknown)}. "
                "A newer plan version may have produced this record."
            )
        return cls(**cls._decode(d))

    @classmethod
    def _decode(cls, d: dict[str, Any]) -> dict[str, Any]:
        return dict(d)


def _encode(pairs: list[tuple[str, Any]]) -> dict[str, Any]:
    out: dict[str, Any] = {}
    for key, value in pairs:
        if isinstance(value, Enum):
            value = value.value
        elif isinstance(value, tuple):
            value = list(value)
        out[key] = value
    return out


@dataclass(frozen=True, slots=True)
class ContextItem(_Strict):
    item_id: str
    resolution: Resolution = Resolution.RAW

    @classmethod
    def _decode(cls, d: dict[str, Any]) -> dict[str, Any]:
        return {**d, "resolution": Resolution(d.get("resolution", Resolution.RAW))}


@dataclass(frozen=True, slots=True)
class HardwareProfile(_Strict):
    profile_id: str
    platform: str
    memory_bytes: int
    accelerator: str
    measurable: tuple[str, ...] = ()

    def can_measure(self, field_name: str) -> bool:
        return field_name in self.measurable

    @classmethod
    def _decode(cls, d: dict[str, Any]) -> dict[str, Any]:
        return {**d, "measurable": tuple(d.get("measurable", ()))}


@dataclass(frozen=True, slots=True)
class TaskProfile(_Strict):
    task_id: str
    question: str
    kind: str
    available_item_ids: tuple[str, ...] = ()

    @classmethod
    def _decode(cls, d: dict[str, Any]) -> dict[str, Any]:
        return {**d, "available_item_ids": tuple(d.get("available_item_ids", ()))}


@dataclass(frozen=True, slots=True)
class Plan(_Strict):
    task_id: str
    model: str
    token_budget: int
    context_items: tuple[ContextItem, ...] = ()
    latency_budget_ms: float | None = None
    cost_budget_usd: float | None = None
    selector_id: str = "unspecified"
    plan_version: int = PLAN_VERSION

    @classmethod
    def _decode(cls, d: dict[str, Any]) -> dict[str, Any]:
        items = tuple(ContextItem.from_dict(i) for i in d.get("context_items", ()))
        return {**d, "context_items": items}


@dataclass(frozen=True, slots=True)
class Outcome(_Strict):
    answer: str
    tokens_in: int
    tokens_out: int
    context_tokens_materialised: int
    wall_ms: float
    executor_id: str
    executor_version: str
    ttft_ms: float | None = None
    memory_peak_bytes: int | None = None
    kv_bytes: int | None = None
    bytes_moved: int | None = None
    energy_j: float | None = None
    cost_usd: float | None = None
    failure_mode: FailureMode | None = None
    outcome_version: int = OUTCOME_VERSION

    @classmethod
    def _decode(cls, d: dict[str, Any]) -> dict[str, Any]:
        mode = d.get("failure_mode")
        return {**d, "failure_mode": FailureMode(mode) if mode else None}


@dataclass(frozen=True, slots=True)
class Verdict(_Strict):
    quality: float
    sufficiency: bool | None = None
    failure_diagnosis: str | None = None
    grader_id: str = "unspecified"


@dataclass(frozen=True, slots=True)
class Record(_Strict):
    run_id: str
    seed: int
    created_at: str
    profile: HardwareProfile
    task: TaskProfile
    plan: Plan
    outcome: Outcome
    verdict: Verdict
    suite_id: str
    suite_version: str
    environment: dict[str, str] = field(default_factory=dict)

    @classmethod
    def _decode(cls, d: dict[str, Any]) -> dict[str, Any]:
        return {
            **d,
            "profile": HardwareProfile.from_dict(d["profile"]),
            "task": TaskProfile.from_dict(d["task"]),
            "plan": Plan.from_dict(d["plan"]),
            "outcome": Outcome.from_dict(d["outcome"]),
            "verdict": Verdict.from_dict(d["verdict"]),
        }
```

```toml
# packages/ration-core/pyproject.toml
[project]
name = "ration-core"
version = "0.0.0"
description = "Contracts, ports and the kernel loop. No I/O and no third party dependencies."
requires-python = ">=3.13"
dependencies = []

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/ration_core"]
```

Add to the root `pyproject.toml`:

```toml
[dependency-groups]
dev = ["pytest>=8", "ruff>=0.6", "import-linter>=2"]
```

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest packages/ration-core/tests/test_contracts.py -v`
Expected: PASS, 6 passed

- [ ] **Step 5: Check the zero dependency rule holds**

Run: `uv run python -c "import ration_core.contracts, sys; print(sorted(m for m in sys.modules if not m.startswith(('_','ration'))))"`
Expected: only stdlib module names. If any third party name appears, remove the import before committing.

- [ ] **Step 6: Commit**

```bash
git add pyproject.toml packages/ration-core
git commit -m "feat: add versioned contracts with strict decoding

Plan, Outcome, Verdict, TaskProfile, HardwareProfile and Record as frozen
dataclasses. Unknown fields are rejected rather than ignored, so a record
written by a later phase fails loudly here instead of being silently
truncated. Unmeasurable outcome fields default to None, never zero.

ration-core has no third party dependencies, which a test asserts."
```

---

### Task 2: Ports, and the import rule that keeps the core clean

**Files:**
- Create: `packages/ration-core/src/ration_core/ports.py`
- Create: `packages/ration-core/tests/test_ports.py`
- Create: `.importlinter`
- Create: `.github/workflows/ci.yml`

**Interfaces:**
- Consumes: everything from Task 1
- Produces: `Executor`, `InformationStore`, `Selector`, `Evaluator`, `Meter`, `Ledger` Protocols with the exact signatures below. Every later task implements one of these.

- [ ] **Step 1: Write the failing test**

```python
# packages/ration-core/tests/test_ports.py
from ration_core.contracts import ContextItem, Outcome, Plan, Resolution, TaskProfile, Verdict
from ration_core.ports import Evaluator, Executor, InformationStore, Selector


class _Store:
    def item_ids(self) -> tuple[str, ...]:
        return ("a",)

    def get(self, item_id: str, resolution: Resolution) -> str:
        return "body"

    def token_count(self, item_id: str, resolution: Resolution) -> int:
        return 1


class _Selector:
    selector_id = "test"

    def select(self, task: TaskProfile, store: InformationStore, token_budget: int):
        return (ContextItem(item_id="a", resolution=Resolution.RAW),)


def test_structural_typing_accepts_a_conforming_object():
    assert isinstance(_Store(), InformationStore)
    assert isinstance(_Selector(), Selector)


def test_structural_typing_rejects_a_non_conforming_object():
    class NotAStore:
        def item_ids(self) -> tuple[str, ...]:
            return ()

    assert not isinstance(NotAStore(), InformationStore)


def test_protocols_are_runtime_checkable_for_every_port():
    for port in (Executor, InformationStore, Selector, Evaluator):
        assert isinstance(object(), port) is False
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest packages/ration-core/tests/test_ports.py -v`
Expected: FAIL with `ImportError: cannot import name 'Executor'`

- [ ] **Step 3: Write minimal implementation**

```python
# packages/ration-core/src/ration_core/ports.py
"""The six ports. Implementations live in the other packages, never here."""

from __future__ import annotations

from typing import Protocol, runtime_checkable

from ration_core.contracts import (
    ContextItem,
    HardwareProfile,
    Outcome,
    Plan,
    Record,
    Resolution,
    TaskProfile,
    Verdict,
)


@runtime_checkable
class InformationStore(Protocol):
    def item_ids(self) -> tuple[str, ...]: ...

    def get(self, item_id: str, resolution: Resolution) -> str: ...

    def token_count(self, item_id: str, resolution: Resolution) -> int: ...


@runtime_checkable
class Selector(Protocol):
    selector_id: str

    def select(
        self, task: TaskProfile, store: InformationStore, token_budget: int
    ) -> tuple[ContextItem, ...]: ...


@runtime_checkable
class Executor(Protocol):
    executor_id: str
    executor_version: str

    def execute(self, plan: Plan, store: InformationStore) -> Outcome: ...


@runtime_checkable
class Evaluator(Protocol):
    grader_id: str

    def evaluate(self, task: TaskProfile, outcome: Outcome) -> Verdict: ...


@runtime_checkable
class Meter(Protocol):
    def profile(self) -> HardwareProfile: ...


@runtime_checkable
class Ledger(Protocol):
    def write(self, record: Record) -> None: ...

    def records(self, *, suite_id: str, profile_id: str) -> tuple[Record, ...]: ...
```

```ini
# .importlinter
[importlinter]
root_packages =
    ration_core
    ration_exec
    ration_eval
    ration_ledger
    ration_cli

[importlinter:contract:core-is-independent]
name = ration_core imports nothing from the other packages
type = forbidden
source_modules =
    ration_core
forbidden_modules =
    ration_exec
    ration_eval
    ration_ledger
    ration_cli

[importlinter:contract:layers]
name = Package layering
type = layers
layers =
    ration_cli
    ration_eval | ration_ledger | ration_exec
    ration_core
```

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: astral-sh/setup-uv@v7
        with:
          enable-cache: true
      - run: uv sync --all-packages
      - name: Lint
        run: uv run ruff check packages/
      - name: Format
        run: uv run ruff format --check packages/
      - name: Import contracts
        run: uv run lint-imports
      - name: Tests
        run: uv run pytest
```

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest packages/ration-core/tests/test_ports.py -v`
Expected: PASS, 3 passed

- [ ] **Step 5: Run the import contract**

Run: `uv run lint-imports`
Expected: `Contracts: 2 kept, 0 broken.`

- [ ] **Step 6: Commit**

```bash
git add packages/ration-core/src/ration_core/ports.py packages/ration-core/tests/test_ports.py .importlinter .github/workflows/ci.yml
git commit -m "feat: define the six ports and enforce the core independence rule

Executor, InformationStore, Selector, Evaluator, Meter and Ledger as
runtime checkable Protocols. Structural typing means a baseline and a
Ration policy are the same kind of object, which is what makes the
comparison in this project fair by construction rather than by care.

import-linter enforces that ration_core imports nothing from the other
packages, and CI fails the build if it does."
```

---

### Task 3: The kernel loop

**Files:**
- Create: `packages/ration-core/src/ration_core/kernel.py`
- Create: `packages/ration-core/tests/test_kernel.py`

**Interfaces:**
- Consumes: contracts and ports from Tasks 1 and 2
- Produces: `run_once(task, store, selector, executor, evaluator, *, model, token_budget, profile, seed, suite_id, suite_version, environment, clock, run_id) -> Record`. Callers in Task 12 and the CLI rely on this exact signature.

- [ ] **Step 1: Write the failing test**

```python
# packages/ration-core/tests/test_kernel.py
import pytest

from ration_core.contracts import (
    ContextItem,
    HardwareProfile,
    Outcome,
    Resolution,
    TaskProfile,
    Verdict,
)
from ration_core.errors import BudgetExceededError
from ration_core.kernel import run_once


class _Store:
    def item_ids(self):
        return ("a", "b")

    def get(self, item_id, resolution):
        return f"{item_id}-{resolution.value}"

    def token_count(self, item_id, resolution):
        return 10


class _Selector:
    selector_id = "fixed"

    def __init__(self, ids):
        self._ids = ids

    def select(self, task, store, token_budget):
        return tuple(ContextItem(item_id=i, resolution=Resolution.RAW) for i in self._ids)


class _Executor:
    executor_id = "double"
    executor_version = "1"

    def execute(self, plan, store):
        return Outcome(
            answer="answer",
            tokens_in=sum(store.token_count(i.item_id, i.resolution) for i in plan.context_items),
            tokens_out=1,
            context_tokens_materialised=sum(
                store.token_count(i.item_id, i.resolution) for i in plan.context_items
            ),
            wall_ms=1.0,
            executor_id=self.executor_id,
            executor_version=self.executor_version,
        )


class _Evaluator:
    grader_id = "always-one"

    def evaluate(self, task, outcome):
        return Verdict(quality=1.0, grader_id=self.grader_id)


def _profile():
    return HardwareProfile(
        profile_id="p1", platform="test", memory_bytes=1, accelerator="none",
        measurable=("tokens_in", "wall_ms"),
    )


def _task():
    return TaskProfile(
        task_id="t1", question="q", kind="definition", available_item_ids=("a", "b")
    )


def _run(selector, budget=100):
    return run_once(
        task=_task(),
        store=_Store(),
        selector=selector,
        executor=_Executor(),
        evaluator=_Evaluator(),
        model="double:v1",
        token_budget=budget,
        profile=_profile(),
        seed=7,
        suite_id="suite",
        suite_version="v1",
        environment={"python": "3.13"},
        run_id="r1",
        now=lambda: "2026-09-22T00:00:00Z",
    )


def test_record_carries_everything_needed_to_reproduce_the_run():
    record = _run(_Selector(["a"]))
    assert record.run_id == "r1"
    assert record.seed == 7
    assert record.suite_id == "suite"
    assert record.suite_version == "v1"
    assert record.profile.profile_id == "p1"
    assert record.environment == {"python": "3.13"}
    assert record.created_at == "2026-09-22T00:00:00Z"


def test_plan_records_which_selector_produced_it():
    assert _run(_Selector(["a"])).plan.selector_id == "fixed"


def test_kernel_refuses_a_selection_over_budget():
    with pytest.raises(BudgetExceededError) as exc:
        _run(_Selector(["a", "b"]), budget=15)
    assert "15" in str(exc.value)


def test_kernel_does_no_io_of_its_own():
    record = _run(_Selector(["a"]))
    assert record.outcome.executor_id == "double"
    assert record.verdict.grader_id == "always-one"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest packages/ration-core/tests/test_kernel.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'ration_core.kernel'`

- [ ] **Step 3: Write minimal implementation**

```python
# packages/ration-core/src/ration_core/kernel.py
"""The loop. Every phase widens what the planner may decide and changes nothing here."""

from __future__ import annotations

from collections.abc import Callable

from ration_core.contracts import HardwareProfile, Plan, Record, TaskProfile
from ration_core.errors import BudgetExceededError
from ration_core.ports import Evaluator, Executor, InformationStore, Selector


def run_once(
    *,
    task: TaskProfile,
    store: InformationStore,
    selector: Selector,
    executor: Executor,
    evaluator: Evaluator,
    model: str,
    token_budget: int,
    profile: HardwareProfile,
    seed: int,
    suite_id: str,
    suite_version: str,
    environment: dict[str, str],
    run_id: str,
    now: Callable[[], str],
) -> Record:
    """Appraise, plan, execute, evaluate, record. Returns the record, writes nothing."""
    items = selector.select(task, store, token_budget)

    materialised = sum(store.token_count(i.item_id, i.resolution) for i in items)
    if materialised > token_budget:
        raise BudgetExceededError(
            f"selector {selector.selector_id!r} chose {materialised} tokens "
            f"against a budget of {token_budget}"
        )

    plan = Plan(
        task_id=task.task_id,
        model=model,
        token_budget=token_budget,
        context_items=items,
        selector_id=selector.selector_id,
    )
    outcome = executor.execute(plan, store)
    verdict = evaluator.evaluate(task, outcome)

    return Record(
        run_id=run_id,
        seed=seed,
        created_at=now(),
        profile=profile,
        task=task,
        plan=plan,
        outcome=outcome,
        verdict=verdict,
        suite_id=suite_id,
        suite_version=suite_version,
        environment=dict(environment),
    )
```

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest packages/ration-core/tests/test_kernel.py -v`
Expected: PASS, 4 passed

- [ ] **Step 5: Commit**

```bash
git add packages/ration-core/src/ration_core/kernel.py packages/ration-core/tests/test_kernel.py
git commit -m "feat: add the kernel loop

Appraise, plan, execute, evaluate, record. The kernel does no I/O, takes
its clock as an argument and returns a record rather than writing one, so
it is testable with no model, no disk and no network.

A selection over budget raises rather than being truncated silently. A
budget that is quietly not enforced would make every later comparison
meaningless."
```

---

### Task 4: Deterministic executor, Apple silicon meter and honest nulls

**Files:**
- Create: `packages/ration-exec/pyproject.toml`
- Create: `packages/ration-exec/src/ration_exec/__init__.py`
- Create: `packages/ration-exec/src/ration_exec/double.py`
- Create: `packages/ration-exec/src/ration_exec/profile.py`
- Create: `packages/ration-exec/src/ration_exec/meter_apple.py`
- Create: `packages/ration-exec/tests/test_double.py`
- Create: `packages/ration-exec/tests/test_meter_apple.py`

**Interfaces:**
- Consumes: `Executor`, `Meter`, `Outcome`, `HardwareProfile`, `Plan`, `InformationStore`
- Produces: `DeterministicExecutor(answers: dict[str, str], *, tokens_per_item: int = 0)` with `executor_id = "double"`, `executor_version = "1"`. `AppleMeter()` with `.profile() -> HardwareProfile`. `count_tokens(text: str) -> int`, the single tokenisation used everywhere in phase 1.

- [ ] **Step 1: Write the failing test**

```python
# packages/ration-exec/tests/test_double.py
from ration_core.contracts import ContextItem, Plan, Resolution

from ration_exec.double import DeterministicExecutor, count_tokens


class _Store:
    def item_ids(self):
        return ("a", "b")

    def get(self, item_id, resolution):
        return f"body of {item_id}"

    def token_count(self, item_id, resolution):
        return count_tokens(self.get(item_id, resolution))


def _plan(*ids):
    return Plan(
        task_id="t1",
        model="double:v1",
        token_budget=1000,
        context_items=tuple(ContextItem(item_id=i, resolution=Resolution.RAW) for i in ids),
    )


def test_same_plan_gives_the_same_answer_every_time():
    ex = DeterministicExecutor(answers={"t1": "the answer"})
    first = ex.execute(_plan("a"), _Store())
    second = ex.execute(_plan("a"), _Store())
    assert first.answer == second.answer == "the answer"


def test_it_reports_only_the_context_it_was_given():
    ex = DeterministicExecutor(answers={"t1": "x"})
    one = ex.execute(_plan("a"), _Store())
    two = ex.execute(_plan("a", "b"), _Store())
    assert two.context_tokens_materialised > one.context_tokens_materialised


def test_unmeasurable_fields_stay_none():
    outcome = DeterministicExecutor(answers={"t1": "x"}).execute(_plan("a"), _Store())
    assert outcome.kv_bytes is None
    assert outcome.bytes_moved is None
    assert outcome.cost_usd is None


def test_a_missing_answer_is_a_failure_mode_not_an_exception():
    outcome = DeterministicExecutor(answers={}).execute(_plan("a"), _Store())
    assert outcome.failure_mode is not None
    assert outcome.answer == ""


def test_token_counting_is_stable_and_whitespace_based():
    assert count_tokens("def f(x):\n    return x") == count_tokens("def f(x):  return x")
```

```python
# packages/ration-exec/tests/test_meter_apple.py
import sys

import pytest

from ration_exec.meter_apple import AppleMeter

pytestmark = pytest.mark.skipif(sys.platform != "darwin", reason="Apple silicon meter")


def test_profile_is_stable_across_calls():
    meter = AppleMeter()
    assert meter.profile().profile_id == meter.profile().profile_id


def test_profile_declares_kv_bytes_unmeasurable():
    profile = AppleMeter().profile()
    assert profile.can_measure("kv_bytes") is False
    assert profile.can_measure("bytes_moved") is False


def test_profile_declares_what_it_can_measure():
    profile = AppleMeter().profile()
    for name in ("tokens_in", "tokens_out", "context_tokens_materialised", "wall_ms"):
        assert profile.can_measure(name) is True


def test_unified_memory_is_recorded_as_the_platform():
    assert "unified" in AppleMeter().profile().platform
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `uv run pytest packages/ration-exec -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'ration_exec'`

- [ ] **Step 3: Write minimal implementation**

```python
# packages/ration-exec/src/ration_exec/double.py
"""A deterministic executor. Every test in this repository runs against it."""

from __future__ import annotations

import time

from ration_core.contracts import FailureMode, Outcome, Plan
from ration_core.ports import InformationStore


def count_tokens(text: str) -> int:
    """The one tokenisation phase 1 uses.

    Whitespace splitting, chosen because it is identical on every machine and
    for every model. It is not a model tokenizer and does not pretend to be:
    every number derived from it is a count of this unit, and the report says so.
    """
    return len(text.split())


class DeterministicExecutor:
    executor_id = "double"
    executor_version = "1"

    def __init__(self, answers: dict[str, str]):
        self._answers = dict(answers)

    def execute(self, plan: Plan, store: InformationStore) -> Outcome:
        started = time.perf_counter()
        materialised = sum(
            store.token_count(item.item_id, item.resolution) for item in plan.context_items
        )
        answer = self._answers.get(plan.task_id, "")
        elapsed_ms = (time.perf_counter() - started) * 1000
        return Outcome(
            answer=answer,
            tokens_in=materialised,
            tokens_out=count_tokens(answer),
            context_tokens_materialised=materialised,
            wall_ms=elapsed_ms,
            ttft_ms=elapsed_ms,
            executor_id=self.executor_id,
            executor_version=self.executor_version,
            failure_mode=None if answer else FailureMode.EMPTY_ANSWER,
        )
```

```python
# packages/ration-exec/src/ration_exec/meter_apple.py
"""Apple silicon meter. What it cannot measure, it declares unmeasurable. See ADR 0004."""

from __future__ import annotations

import platform
import subprocess

from ration_core.contracts import HardwareProfile

_MEASURABLE = (
    "tokens_in",
    "tokens_out",
    "context_tokens_materialised",
    "wall_ms",
    "ttft_ms",
    "memory_peak_bytes",
)


def _sysctl(name: str) -> str:
    try:
        return subprocess.run(
            ["sysctl", "-n", name], capture_output=True, text=True, check=True, timeout=5
        ).stdout.strip()
    except (subprocess.SubprocessError, OSError):
        return "unknown"


class AppleMeter:
    """Reports what this machine is and, more importantly, what it cannot tell you.

    kv_bytes and bytes_moved are absent on purpose. Unified memory has no
    separate accelerator memory to move bytes into, and Ollama runs in another
    process and does not expose its cache size. Both stay None in every record.
    """

    def profile(self) -> HardwareProfile:
        chip = _sysctl("machdep.cpu.brand_string")
        raw_memory = _sysctl("hw.memsize")
        memory_bytes = int(raw_memory) if raw_memory.isdigit() else 0
        return HardwareProfile(
            profile_id=f"apple-{chip}-{memory_bytes}-{platform.mac_ver()[0]}".replace(" ", "-"),
            platform="apple-silicon-unified-memory",
            memory_bytes=memory_bytes,
            accelerator=chip,
            measurable=_MEASURABLE,
        )
```

```toml
# packages/ration-exec/pyproject.toml
[project]
name = "ration-exec"
version = "0.0.0"
description = "Executor adapters and hardware meters."
requires-python = ">=3.13"
dependencies = ["ration-core"]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/ration_exec"]

[tool.uv.sources]
ration-core = { workspace = true }
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `uv run pytest packages/ration-exec -v`
Expected: PASS, 9 passed on macOS. The meter tests skip on other platforms.

- [ ] **Step 5: Commit**

```bash
git add packages/ration-exec
git commit -m "feat: add the deterministic executor and the Apple silicon meter

The double makes every test in this repository runnable with no model, no
network and no API key. Token counting is whitespace based on purpose: it is
identical on every machine, and the report states the unit rather than
implying it is a model tokenizer.

The meter declares kv_bytes and bytes_moved unmeasurable on unified memory
rather than estimating them. That is ADR 0004 in its first concrete form."
```

---

### Task 5: Ollama executor adapter

**Files:**
- Create: `packages/ration-exec/src/ration_exec/ollama.py`
- Create: `packages/ration-exec/tests/test_ollama.py`

**Interfaces:**
- Consumes: `Executor`, `Outcome`, `Plan`, `InformationStore`, `count_tokens`
- Produces: `OllamaExecutor(model: str, *, host: str = "http://127.0.0.1:11434", timeout_s: float = 300, transport: Callable[[str, dict], dict] | None = None)`. `executor_id = "ollama"`. The `transport` argument exists so tests never open a socket.

- [ ] **Step 1: Write the failing test**

```python
# packages/ration-exec/tests/test_ollama.py
import pytest
from ration_core.contracts import ContextItem, FailureMode, Plan, Resolution

from ration_exec.ollama import OllamaExecutor


class _Store:
    def item_ids(self):
        return ("a",)

    def get(self, item_id, resolution):
        return "def f(x):\n    return x"

    def token_count(self, item_id, resolution):
        return 4


def _plan():
    return Plan(
        task_id="t1",
        model="qwen3:4b",
        token_budget=100,
        context_items=(ContextItem(item_id="a", resolution=Resolution.RAW),),
    )


def test_it_sends_the_selected_context_and_nothing_else():
    seen = {}

    def transport(url, body):
        seen["url"] = url
        seen["body"] = body
        return {"response": "f", "prompt_eval_count": 12, "eval_count": 1}

    OllamaExecutor("qwen3:4b", transport=transport).execute(_plan(), _Store())
    assert "def f(x)" in seen["body"]["prompt"]
    assert seen["body"]["stream"] is False
    assert seen["body"]["options"]["seed"] is not None


def test_it_prefers_the_engine_token_counts_over_its_own():
    def transport(url, body):
        return {"response": "f", "prompt_eval_count": 999, "eval_count": 7}

    outcome = OllamaExecutor("qwen3:4b", transport=transport).execute(_plan(), _Store())
    assert outcome.tokens_in == 999
    assert outcome.tokens_out == 7


def test_context_tokens_materialised_uses_the_repository_unit_not_the_engine_unit():
    def transport(url, body):
        return {"response": "f", "prompt_eval_count": 999, "eval_count": 7}

    outcome = OllamaExecutor("qwen3:4b", transport=transport).execute(_plan(), _Store())
    assert outcome.context_tokens_materialised == 4


def test_kv_bytes_stays_none_because_ollama_does_not_report_it():
    def transport(url, body):
        return {"response": "f", "prompt_eval_count": 1, "eval_count": 1}

    outcome = OllamaExecutor("qwen3:4b", transport=transport).execute(_plan(), _Store())
    assert outcome.kv_bytes is None
    assert outcome.memory_peak_bytes is None


def test_a_transport_failure_becomes_a_failure_mode_not_a_crash():
    def transport(url, body):
        raise OSError("connection refused")

    outcome = OllamaExecutor("qwen3:4b", transport=transport).execute(_plan(), _Store())
    assert outcome.failure_mode is FailureMode.EXECUTOR_ERROR
    assert outcome.answer == ""
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest packages/ration-exec/tests/test_ollama.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'ration_exec.ollama'`

- [ ] **Step 3: Write minimal implementation**

```python
# packages/ration-exec/src/ration_exec/ollama.py
"""Ollama adapter. Uses urllib so ration-exec stays dependency light."""

from __future__ import annotations

import json
import time
import urllib.request
from collections.abc import Callable

from ration_core.contracts import FailureMode, Outcome, Plan
from ration_core.ports import InformationStore

from ration_exec.double import count_tokens

PROMPT_TEMPLATE = """Answer the question using only the context below.
Answer with the shortest correct answer and nothing else.

Context:
{context}

Question: {question}
Answer:"""


def _http(url: str, body: dict) -> dict:
    request = urllib.request.Request(
        url,
        data=json.dumps(body).encode(),
        headers={"Content-Type": "application/json"},
        method="POST",
    )
    with urllib.request.urlopen(request, timeout=body.pop("_timeout_s", 300)) as response:
        return json.loads(response.read())


class OllamaExecutor:
    executor_id = "ollama"

    def __init__(
        self,
        model: str,
        *,
        host: str = "http://127.0.0.1:11434",
        timeout_s: float = 300,
        seed: int = 0,
        transport: Callable[[str, dict], dict] | None = None,
    ):
        self._model = model
        self._host = host.rstrip("/")
        self._timeout_s = timeout_s
        self._seed = seed
        self._transport = transport or _http
        self.executor_version = model

    def execute(self, plan: Plan, store: InformationStore) -> Outcome:
        context = "\n\n".join(
            store.get(item.item_id, item.resolution) for item in plan.context_items
        )
        materialised = sum(
            store.token_count(item.item_id, item.resolution) for item in plan.context_items
        )
        question = plan.task_id if not context else plan.task_id
        prompt = PROMPT_TEMPLATE.format(context=context, question=question)

        started = time.perf_counter()
        try:
            response = self._transport(
                f"{self._host}/api/generate",
                {
                    "model": self._model,
                    "prompt": prompt,
                    "stream": False,
                    "options": {"seed": self._seed, "temperature": 0.0},
                    "_timeout_s": self._timeout_s,
                },
            )
        except (OSError, ValueError):
            return Outcome(
                answer="",
                tokens_in=materialised,
                tokens_out=0,
                context_tokens_materialised=materialised,
                wall_ms=(time.perf_counter() - started) * 1000,
                executor_id=self.executor_id,
                executor_version=self.executor_version,
                failure_mode=FailureMode.EXECUTOR_ERROR,
            )

        answer = str(response.get("response", "")).strip()
        return Outcome(
            answer=answer,
            tokens_in=int(response.get("prompt_eval_count") or count_tokens(prompt)),
            tokens_out=int(response.get("eval_count") or count_tokens(answer)),
            context_tokens_materialised=materialised,
            wall_ms=(time.perf_counter() - started) * 1000,
            ttft_ms=_ns_to_ms(response.get("prompt_eval_duration")),
            executor_id=self.executor_id,
            executor_version=self.executor_version,
            failure_mode=None if answer else FailureMode.EMPTY_ANSWER,
        )


def _ns_to_ms(value: object) -> float | None:
    return float(value) / 1e6 if isinstance(value, int | float) else None
```

**Note for the implementer:** `plan.task_id` is not the question text. The plan carries an id, and the question lives on the `TaskProfile`. Task 12 threads the question through by constructing the executor per task with the question bound, or by adding a `question` field to `Plan`. Choose the second: add `question: str = ""` to `Plan` in `contracts.py`, extend `test_contracts.py` to cover it, and use `plan.question` here. Do not leave the `task_id` fallback in place.

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest packages/ration-exec/tests/test_ollama.py -v`
Expected: PASS, 5 passed

- [ ] **Step 5: Verify it works against a real Ollama, if one is running**

Run: `uv run python -c "
from ration_exec.ollama import OllamaExecutor
print(OllamaExecutor('qwen3:4b').execute.__doc__ or 'adapter importable')
"`
Expected: no error. A live check belongs in Task 14, not here.

- [ ] **Step 6: Commit**

```bash
git add packages/ration-exec/src/ration_exec/ollama.py packages/ration-exec/tests/test_ollama.py packages/ration-core
git commit -m "feat: add the Ollama executor adapter

Transport is injectable, so every test runs without opening a socket.

The engine's own prompt_eval_count and eval_count are preferred for
tokens_in and tokens_out, while context_tokens_materialised keeps this
repository's own unit. Mixing the two would make the frontier
incomparable across executors, which is the whole point of the axis.

kv_bytes and memory_peak_bytes stay None: Ollama is a separate process and
reports neither. A transport failure becomes a failure mode on the record
rather than an exception, so a failed run is still a measured run."
```

---

### Task 6: The ledger, and the refusal that keeps results honest

**Files:**
- Create: `packages/ration-ledger/pyproject.toml`
- Create: `packages/ration-ledger/src/ration_ledger/__init__.py`
- Create: `packages/ration-ledger/src/ration_ledger/schema.py`
- Create: `packages/ration-ledger/src/ration_ledger/store.py`
- Create: `packages/ration-ledger/tests/test_store.py`

**Interfaces:**
- Consumes: `Record`, `Ledger`, `ProfileMismatchError`
- Produces: `SqliteLedger(path: str | Path)` with `.write(record)`, `.records(*, suite_id, profile_id)`, `.profiles() -> tuple[str, ...]`, `.suites() -> tuple[str, ...]`, and `.comparable(*, suite_id, selector_ids, profile_id) -> dict[str, tuple[Record, ...]]` which raises `ProfileMismatchError` if asked to mix profiles.

- [ ] **Step 1: Write the failing test**

```python
# packages/ration-ledger/tests/test_store.py
import pytest
from ration_core.contracts import (
    ContextItem,
    HardwareProfile,
    Outcome,
    Plan,
    Record,
    Resolution,
    TaskProfile,
    Verdict,
)
from ration_core.errors import ProfileMismatchError

from ration_ledger.store import SqliteLedger


def _record(*, run_id="r1", profile_id="p1", selector="ration", quality=1.0):
    return Record(
        run_id=run_id,
        seed=7,
        created_at="2026-09-22T00:00:00Z",
        profile=HardwareProfile(
            profile_id=profile_id, platform="test", memory_bytes=1,
            accelerator="none", measurable=("wall_ms",),
        ),
        task=TaskProfile(task_id="t1", question="q", kind="definition"),
        plan=Plan(
            task_id="t1", model="m", token_budget=100, selector_id=selector,
            context_items=(ContextItem(item_id="a", resolution=Resolution.RAW),),
        ),
        outcome=Outcome(
            answer="a", tokens_in=1, tokens_out=1, context_tokens_materialised=1,
            wall_ms=1.0, executor_id="double", executor_version="1",
        ),
        verdict=Verdict(quality=quality, grader_id="g"),
        suite_id="code-understanding",
        suite_version="v1",
        environment={"python": "3.13"},
    )


def test_a_written_record_round_trips_unchanged(tmp_path):
    ledger = SqliteLedger(tmp_path / "l.db")
    original = _record()
    ledger.write(original)
    stored = ledger.records(suite_id="code-understanding", profile_id="p1")
    assert stored == (original,)


def test_writing_the_same_run_id_twice_is_rejected(tmp_path):
    ledger = SqliteLedger(tmp_path / "l.db")
    ledger.write(_record())
    with pytest.raises(Exception):
        ledger.write(_record())


def test_comparing_across_hardware_profiles_is_refused(tmp_path):
    ledger = SqliteLedger(tmp_path / "l.db")
    ledger.write(_record(run_id="r1", profile_id="p1", selector="ration"))
    ledger.write(_record(run_id="r2", profile_id="p2", selector="full-context"))
    with pytest.raises(ProfileMismatchError) as exc:
        ledger.comparable(
            suite_id="code-understanding",
            selector_ids=("ration", "full-context"),
            profile_id=None,
        )
    assert "p1" in str(exc.value) and "p2" in str(exc.value)


def test_comparison_within_one_profile_returns_a_group_per_selector(tmp_path):
    ledger = SqliteLedger(tmp_path / "l.db")
    ledger.write(_record(run_id="r1", selector="ration"))
    ledger.write(_record(run_id="r2", selector="full-context"))
    groups = ledger.comparable(
        suite_id="code-understanding",
        selector_ids=("ration", "full-context"),
        profile_id="p1",
    )
    assert set(groups) == {"ration", "full-context"}
    assert len(groups["ration"]) == 1


def test_a_selector_with_no_runs_is_reported_rather_than_silently_missing(tmp_path):
    ledger = SqliteLedger(tmp_path / "l.db")
    ledger.write(_record(run_id="r1", selector="ration"))
    groups = ledger.comparable(
        suite_id="code-understanding", selector_ids=("ration", "oracle"), profile_id="p1"
    )
    assert groups["oracle"] == ()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest packages/ration-ledger -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'ration_ledger'`

- [ ] **Step 3: Write minimal implementation**

```python
# packages/ration-ledger/src/ration_ledger/schema.py
SCHEMA = """
CREATE TABLE IF NOT EXISTS runs (
    run_id        TEXT PRIMARY KEY,
    suite_id      TEXT NOT NULL,
    suite_version TEXT NOT NULL,
    profile_id    TEXT NOT NULL,
    selector_id   TEXT NOT NULL,
    task_id       TEXT NOT NULL,
    created_at    TEXT NOT NULL,
    quality       REAL NOT NULL,
    materialised  INTEGER NOT NULL,
    wall_ms       REAL NOT NULL,
    record_json   TEXT NOT NULL
);
CREATE INDEX IF NOT EXISTS runs_by_group ON runs (suite_id, profile_id, selector_id);
"""
```

```python
# packages/ration-ledger/src/ration_ledger/store.py
"""The ledger. The only place a number in this repository may come from."""

from __future__ import annotations

import json
import sqlite3
from pathlib import Path

from ration_core.contracts import Record
from ration_core.errors import ProfileMismatchError

from ration_ledger.schema import SCHEMA


class SqliteLedger:
    def __init__(self, path: str | Path):
        self._path = Path(path)
        self._path.parent.mkdir(parents=True, exist_ok=True)
        with self._connect() as conn:
            conn.executescript(SCHEMA)

    def _connect(self) -> sqlite3.Connection:
        conn = sqlite3.connect(self._path)
        conn.row_factory = sqlite3.Row
        return conn

    def write(self, record: Record) -> None:
        with self._connect() as conn:
            conn.execute(
                "INSERT INTO runs (run_id, suite_id, suite_version, profile_id, selector_id,"
                " task_id, created_at, quality, materialised, wall_ms, record_json)"
                " VALUES (?,?,?,?,?,?,?,?,?,?,?)",
                (
                    record.run_id,
                    record.suite_id,
                    record.suite_version,
                    record.profile.profile_id,
                    record.plan.selector_id,
                    record.task.task_id,
                    record.created_at,
                    record.verdict.quality,
                    record.outcome.context_tokens_materialised,
                    record.outcome.wall_ms,
                    json.dumps(record.to_dict()),
                ),
            )

    def records(self, *, suite_id: str, profile_id: str) -> tuple[Record, ...]:
        with self._connect() as conn:
            rows = conn.execute(
                "SELECT record_json FROM runs WHERE suite_id = ? AND profile_id = ?"
                " ORDER BY created_at, run_id",
                (suite_id, profile_id),
            ).fetchall()
        return tuple(Record.from_dict(json.loads(r["record_json"])) for r in rows)

    def profiles(self) -> tuple[str, ...]:
        with self._connect() as conn:
            rows = conn.execute("SELECT DISTINCT profile_id FROM runs ORDER BY 1").fetchall()
        return tuple(r["profile_id"] for r in rows)

    def suites(self) -> tuple[str, ...]:
        with self._connect() as conn:
            rows = conn.execute("SELECT DISTINCT suite_id FROM runs ORDER BY 1").fetchall()
        return tuple(r["suite_id"] for r in rows)

    def comparable(
        self, *, suite_id: str, selector_ids: tuple[str, ...], profile_id: str | None
    ) -> dict[str, tuple[Record, ...]]:
        """Group runs by selector for one profile.

        Refuses to mix profiles. Two machines produce two sets of numbers that
        look like one, and nothing downstream can tell them apart. See ADR 0004.
        """
        placeholders = ",".join("?" for _ in selector_ids)
        with self._connect() as conn:
            found = conn.execute(
                f"SELECT DISTINCT profile_id FROM runs WHERE suite_id = ?"
                f" AND selector_id IN ({placeholders}) ORDER BY 1",
                (suite_id, *selector_ids),
            ).fetchall()
        profile_ids = [r["profile_id"] for r in found]

        if profile_id is None:
            if len(profile_ids) > 1:
                raise ProfileMismatchError(
                    "these runs span more than one hardware profile and cannot be compared: "
                    + ", ".join(profile_ids)
                    + ". Pass profile_id to choose one."
                )
            if not profile_ids:
                return {sid: () for sid in selector_ids}
            profile_id = profile_ids[0]

        rows = self.records(suite_id=suite_id, profile_id=profile_id)
        return {
            sid: tuple(r for r in rows if r.plan.selector_id == sid) for sid in selector_ids
        }
```

```toml
# packages/ration-ledger/pyproject.toml
[project]
name = "ration-ledger"
version = "0.0.0"
description = "Persistence, hardware profiles and report generation."
requires-python = ">=3.13"
dependencies = ["ration-core"]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/ration_ledger"]

[tool.uv.sources]
ration-core = { workspace = true }
```

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest packages/ration-ledger -v`
Expected: PASS, 5 passed

- [ ] **Step 5: Commit**

```bash
git add packages/ration-ledger
git commit -m "feat: add the SQLite ledger and refuse cross profile comparison

Every run is stored whole as JSON, with the fields needed for grouping
denormalised into columns. A record round trips unchanged, which is what
makes reproduction from the ledger alone possible.

comparable() raises rather than returning a mixed set when runs span more
than one hardware profile. Two machines produce two sets of numbers that
look like one, and nothing downstream can tell them apart. A selector with
no runs comes back as an empty group rather than a missing key, so a
missing baseline shows up in the report instead of disappearing."
```

---

### Task 7: Repository corpus as an InformationStore

**Files:**
- Create: `packages/ration-eval/pyproject.toml`
- Create: `packages/ration-eval/src/ration_eval/__init__.py`
- Create: `packages/ration-eval/src/ration_eval/corpus.py`
- Create: `tests/fixtures/minirepo/` (see step 1)
- Create: `packages/ration-eval/tests/test_corpus.py`

**Interfaces:**
- Consumes: `InformationStore`, `Resolution`, `count_tokens`
- Produces: `RepoCorpus.from_path(root: Path) -> RepoCorpus` implementing `InformationStore`, with items keyed `"<relative/path>::<qualified_name>"`. Also `.symbols() -> dict[str, Symbol]` where `Symbol` has `name`, `qualname`, `path`, `lineno`, `kind` (`"function"` or `"class"`), `params: tuple[str, ...]`, `calls: frozenset[str]`, `imports: frozenset[str]`. Task 8 builds ground truth from `.symbols()`.

- [ ] **Step 1: Create the fixture repository**

```bash
mkdir -p tests/fixtures/minirepo/pkg
cat > tests/fixtures/minirepo/pkg/__init__.py <<'EOF'
EOF
cat > tests/fixtures/minirepo/pkg/maths.py <<'EOF'
def add(left, right):
    return left + right


def double(value):
    return add(value, value)
EOF
cat > tests/fixtures/minirepo/pkg/app.py <<'EOF'
from pkg.maths import double


class Runner:
    def run(self, value):
        return double(value)
EOF
```

- [ ] **Step 2: Write the failing test**

```python
# packages/ration-eval/tests/test_corpus.py
from pathlib import Path

from ration_core.contracts import Resolution

from ration_eval.corpus import RepoCorpus

FIXTURE = Path(__file__).parents[3] / "tests" / "fixtures" / "minirepo"


def _corpus() -> RepoCorpus:
    return RepoCorpus.from_path(FIXTURE)


def test_every_function_and_class_becomes_one_item():
    ids = set(_corpus().item_ids())
    assert "pkg/maths.py::add" in ids
    assert "pkg/maths.py::double" in ids
    assert "pkg/app.py::Runner.run" in ids


def test_an_item_body_is_the_source_of_that_symbol_only():
    body = _corpus().get("pkg/maths.py::add", Resolution.RAW)
    assert "def add(left, right)" in body
    assert "def double" not in body


def test_token_count_matches_the_body_it_would_return():
    corpus = _corpus()
    body = corpus.get("pkg/maths.py::add", Resolution.RAW)
    assert corpus.token_count("pkg/maths.py::add", Resolution.RAW) == len(body.split())


def test_symbols_record_where_each_one_is_defined():
    symbol = _corpus().symbols()["pkg/maths.py::add"]
    assert symbol.path == "pkg/maths.py"
    assert symbol.kind == "function"
    assert symbol.params == ("left", "right")


def test_symbols_record_who_calls_whom():
    assert "add" in _corpus().symbols()["pkg/maths.py::double"].calls
    assert "double" in _corpus().symbols()["pkg/app.py::Runner.run"].calls


def test_symbols_record_imports():
    assert "pkg.maths.double" in _corpus().symbols()["pkg/app.py::Runner.run"].imports


def test_a_syntactically_broken_file_is_skipped_rather_than_failing_the_corpus(tmp_path):
    (tmp_path / "broken.py").write_text("def (:\n")
    (tmp_path / "fine.py").write_text("def ok():\n    return 1\n")
    assert _corpus_ids(tmp_path) == ("fine.py::ok",)


def _corpus_ids(root):
    return RepoCorpus.from_path(root).item_ids()
```

- [ ] **Step 3: Run test to verify it fails**

Run: `uv run pytest packages/ration-eval/tests/test_corpus.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'ration_eval'`

- [ ] **Step 4: Write minimal implementation**

```python
# packages/ration-eval/src/ration_eval/corpus.py
"""A repository as an InformationStore, chunked one item per function or class method.

Chosen because it is the honest version of the phase 1 problem: the available
information vastly exceeds any context window, the units are natural rather
than arbitrary character windows, and ground truth is derivable exactly from
the syntax tree rather than from a judge.
"""

from __future__ import annotations

import ast
from dataclasses import dataclass
from pathlib import Path

from ration_core.contracts import Resolution

SKIP_DIRS = {".git", ".venv", "__pycache__", "node_modules", "build", "dist", ".ruff_cache"}


@dataclass(frozen=True, slots=True)
class Symbol:
    qualname: str
    name: str
    path: str
    lineno: int
    kind: str
    params: tuple[str, ...]
    calls: frozenset[str]
    imports: frozenset[str]


class RepoCorpus:
    def __init__(self, bodies: dict[str, str], symbols: dict[str, Symbol]):
        self._bodies = bodies
        self._symbols = symbols

    @classmethod
    def from_path(cls, root: Path) -> RepoCorpus:
        bodies: dict[str, str] = {}
        symbols: dict[str, Symbol] = {}
        root = Path(root)
        for file in sorted(root.rglob("*.py")):
            if any(part in SKIP_DIRS for part in file.relative_to(root).parts):
                continue
            source = file.read_text(encoding="utf-8", errors="replace")
            try:
                tree = ast.parse(source)
            except SyntaxError:
                continue
            rel = file.relative_to(root).as_posix()
            imports = _imports(tree)
            for node, qualname in _definitions(tree):
                item_id = f"{rel}::{qualname}"
                bodies[item_id] = ast.get_source_segment(source, node) or ""
                symbols[item_id] = Symbol(
                    qualname=qualname,
                    name=node.name,
                    path=rel,
                    lineno=node.lineno,
                    kind="class" if isinstance(node, ast.ClassDef) else "function",
                    params=_params(node),
                    calls=frozenset(_calls(node)),
                    imports=frozenset(imports),
                )
        return cls(bodies, symbols)

    def item_ids(self) -> tuple[str, ...]:
        return tuple(sorted(self._bodies))

    def get(self, item_id: str, resolution: Resolution) -> str:
        if resolution is not Resolution.RAW:
            raise ValueError(f"phase 1 stores only raw resolution, not {resolution.value}")
        return self._bodies[item_id]

    def token_count(self, item_id: str, resolution: Resolution) -> int:
        return len(self.get(item_id, resolution).split())

    def symbols(self) -> dict[str, Symbol]:
        return dict(self._symbols)


def _definitions(tree: ast.AST, prefix: str = ""):
    for node in ast.iter_child_nodes(tree):
        if isinstance(node, ast.ClassDef):
            qualname = f"{prefix}{node.name}"
            yield node, qualname
            yield from _definitions(node, prefix=f"{qualname}.")
        elif isinstance(node, ast.FunctionDef | ast.AsyncFunctionDef):
            yield node, f"{prefix}{node.name}"


def _params(node: ast.AST) -> tuple[str, ...]:
    if not isinstance(node, ast.FunctionDef | ast.AsyncFunctionDef):
        return ()
    args = node.args
    names = [a.arg for a in (*args.posonlyargs, *args.args, *args.kwonlyargs)]
    return tuple(n for n in names if n not in {"self", "cls"})


def _calls(node: ast.AST) -> set[str]:
    found: set[str] = set()
    for child in ast.walk(node):
        if isinstance(child, ast.Call):
            target = child.func
            if isinstance(target, ast.Name):
                found.add(target.id)
            elif isinstance(target, ast.Attribute):
                found.add(target.attr)
    return found


def _imports(tree: ast.AST) -> set[str]:
    found: set[str] = set()
    for node in ast.walk(tree):
        if isinstance(node, ast.ImportFrom) and node.module:
            found.update(f"{node.module}.{alias.name}" for alias in node.names)
        elif isinstance(node, ast.Import):
            found.update(alias.name for alias in node.names)
    return found
```

```toml
# packages/ration-eval/pyproject.toml
[project]
name = "ration-eval"
version = "0.0.0"
description = "Evaluators, baselines, task suites and metrics."
requires-python = ">=3.13"
dependencies = ["ration-core", "ration-exec"]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/ration_eval"]

[tool.uv.sources]
ration-core = { workspace = true }
ration-exec = { workspace = true }
```

- [ ] **Step 5: Run test to verify it passes**

Run: `uv run pytest packages/ration-eval/tests/test_corpus.py -v`
Expected: PASS, 7 passed

- [ ] **Step 6: Commit**

```bash
git add packages/ration-eval tests/fixtures/minirepo
git commit -m "feat: turn a Python repository into an information store

One item per function, method or class, keyed path::qualified-name, with
the syntax tree kept alongside so ground truth can be derived exactly
rather than judged.

A file that does not parse is skipped rather than failing the corpus.
Real repositories contain generated and vendored files that do not parse,
and a corpus that refuses to load is a corpus nobody measures against."
```

---

### Task 8: Task suite with exact ground truth, and the oracle item set

**Files:**
- Create: `packages/ration-eval/src/ration_eval/suite.py`
- Create: `packages/ration-eval/tests/test_suite.py`

**Interfaces:**
- Consumes: `RepoCorpus`, `Symbol`, `TaskProfile`
- Produces: `build_suite(corpus, *, seed: int, limit: int | None = None) -> Suite`. `Suite` has `suite_id: str`, `suite_version: str`, `tasks: tuple[SuiteTask, ...]`, `.to_dict()` and `Suite.from_dict()`. `SuiteTask` has `task: TaskProfile`, `answer: frozenset[str]`, `oracle_item_ids: tuple[str, ...]`. Question kinds are exactly `"definition_file"`, `"signature"`, `"callers"`, `"import_source"`.

- [ ] **Step 1: Write the failing test**

```python
# packages/ration-eval/tests/test_suite.py
from pathlib import Path

from ration_eval.corpus import RepoCorpus
from ration_eval.suite import Suite, build_suite

FIXTURE = Path(__file__).parents[3] / "tests" / "fixtures" / "minirepo"


def _suite(seed: int = 7) -> Suite:
    return build_suite(RepoCorpus.from_path(FIXTURE), seed=seed)


def test_the_same_seed_produces_the_same_suite():
    assert _suite(7).to_dict() == _suite(7).to_dict()


def test_a_different_seed_produces_a_different_ordering():
    assert _suite(7).to_dict() != _suite(99).to_dict()


def test_every_task_has_a_non_empty_answer_and_an_oracle_set():
    for item in _suite().tasks:
        assert item.answer, item.task.task_id
        assert item.oracle_item_ids, item.task.task_id


def test_the_oracle_items_all_exist_in_the_corpus():
    corpus = RepoCorpus.from_path(FIXTURE)
    available = set(corpus.item_ids())
    for item in build_suite(corpus, seed=7).tasks:
        assert set(item.oracle_item_ids) <= available


def test_definition_file_questions_answer_with_the_path():
    task = _first("definition_file")
    assert task.answer == frozenset({"pkg/maths.py"}) or "pkg/" in next(iter(task.answer))


def test_signature_questions_answer_with_ordered_parameters():
    corpus = RepoCorpus.from_path(FIXTURE)
    for item in build_suite(corpus, seed=7).tasks:
        if item.task.kind == "signature" and "add" in item.task.question:
            assert item.answer == frozenset({"left, right"})
            return
    raise AssertionError("no signature question for add was generated")


def test_callers_questions_answer_with_the_set_of_calling_symbols():
    corpus = RepoCorpus.from_path(FIXTURE)
    for item in build_suite(corpus, seed=7).tasks:
        if item.task.kind == "callers" and item.task.question.endswith("`add`?"):
            assert "double" in {a.split(".")[-1] for a in item.answer}
            return
    raise AssertionError("no callers question for add was generated")


def test_suite_round_trips_through_dict():
    suite = _suite()
    assert Suite.from_dict(suite.to_dict()).to_dict() == suite.to_dict()


def test_available_item_ids_is_the_whole_corpus_not_the_oracle():
    corpus = RepoCorpus.from_path(FIXTURE)
    suite = build_suite(corpus, seed=7)
    assert set(suite.tasks[0].task.available_item_ids) == set(corpus.item_ids())


def _first(kind: str):
    for item in _suite().tasks:
        if item.task.kind == kind:
            return item
    raise AssertionError(f"no {kind} question was generated")
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest packages/ration-eval/tests/test_suite.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'ration_eval.suite'`

- [ ] **Step 3: Write minimal implementation**

```python
# packages/ration-eval/src/ration_eval/suite.py
"""Code understanding tasks with ground truth derived from the syntax tree.

Four question kinds, all answerable exactly and all requiring a small number
of specific items out of the whole repository. No language model grades
anything here, which is why the suite can run offline and why the numbers do
not inherit a judge's variance.
"""

from __future__ import annotations

import random
from dataclasses import dataclass
from typing import Any, Self

from ration_core.contracts import TaskProfile

from ration_eval.corpus import RepoCorpus

SUITE_ID = "code-understanding"
SUITE_VERSION = "v1"


@dataclass(frozen=True, slots=True)
class SuiteTask:
    task: TaskProfile
    answer: frozenset[str]
    oracle_item_ids: tuple[str, ...]

    def to_dict(self) -> dict[str, Any]:
        return {
            "task": self.task.to_dict(),
            "answer": sorted(self.answer),
            "oracle_item_ids": list(self.oracle_item_ids),
        }

    @classmethod
    def from_dict(cls, d: dict[str, Any]) -> Self:
        return cls(
            task=TaskProfile.from_dict(d["task"]),
            answer=frozenset(d["answer"]),
            oracle_item_ids=tuple(d["oracle_item_ids"]),
        )


@dataclass(frozen=True, slots=True)
class Suite:
    suite_id: str
    suite_version: str
    seed: int
    tasks: tuple[SuiteTask, ...]

    def to_dict(self) -> dict[str, Any]:
        return {
            "suite_id": self.suite_id,
            "suite_version": self.suite_version,
            "seed": self.seed,
            "tasks": [t.to_dict() for t in self.tasks],
        }

    @classmethod
    def from_dict(cls, d: dict[str, Any]) -> Self:
        return cls(
            suite_id=d["suite_id"],
            suite_version=d["suite_version"],
            seed=int(d["seed"]),
            tasks=tuple(SuiteTask.from_dict(t) for t in d["tasks"]),
        )


def build_suite(corpus: RepoCorpus, *, seed: int, limit: int | None = None) -> Suite:
    symbols = corpus.symbols()
    all_item_ids = corpus.item_ids()
    callers = _caller_index(symbols)
    tasks: list[SuiteTask] = []

    for item_id, symbol in sorted(symbols.items()):
        if symbol.kind == "function":
            tasks.append(
                _task(
                    f"def-{item_id}",
                    f"Which file defines `{symbol.qualname}`?",
                    "definition_file",
                    {symbol.path},
                    (item_id,),
                    all_item_ids,
                )
            )
            if symbol.params:
                tasks.append(
                    _task(
                        f"sig-{item_id}",
                        f"List the parameters of `{symbol.qualname}` in order, "
                        "comma separated.",
                        "signature",
                        {", ".join(symbol.params)},
                        (item_id,),
                        all_item_ids,
                    )
                )
            if callers.get(symbol.name):
                calling = callers[symbol.name]
                tasks.append(
                    _task(
                        f"call-{item_id}",
                        f"Which functions call `{symbol.name}`?",
                        "callers",
                        {symbols[c].qualname for c in calling},
                        tuple(sorted(calling)),
                        all_item_ids,
                    )
                )
            for imported in sorted(symbol.imports):
                if imported.rsplit(".", 1)[-1] in symbol.calls:
                    tasks.append(
                        _task(
                            f"imp-{item_id}-{imported}",
                            f"Which module does `{symbol.qualname}` import "
                            f"`{imported.rsplit('.', 1)[-1]}` from?",
                            "import_source",
                            {imported.rsplit(".", 1)[0]},
                            (item_id,),
                            all_item_ids,
                        )
                    )

    rng = random.Random(seed)
    rng.shuffle(tasks)
    if limit is not None:
        tasks = tasks[:limit]
    return Suite(SUITE_ID, SUITE_VERSION, seed, tuple(tasks))


def _task(task_id, question, kind, answer, oracle, all_item_ids) -> SuiteTask:
    return SuiteTask(
        task=TaskProfile(
            task_id=task_id,
            question=question,
            kind=kind,
            available_item_ids=tuple(all_item_ids),
        ),
        answer=frozenset(answer),
        oracle_item_ids=tuple(oracle),
    )


def _caller_index(symbols: dict) -> dict[str, set[str]]:
    index: dict[str, set[str]] = {}
    for item_id, symbol in symbols.items():
        for called in symbol.calls:
            index.setdefault(called, set()).add(item_id)
    return {name: {c for c in callers} for name, callers in index.items()}
```

**Note for the implementer:** `_caller_index` includes self recursive calls. Before committing, exclude the symbol from its own caller set, and add a test with a recursive function asserting it is not listed as its own caller. A task whose ground truth is wrong poisons every number derived from it.

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest packages/ration-eval/tests/test_suite.py -v`
Expected: PASS, 10 passed

- [ ] **Step 5: Commit**

```bash
git add packages/ration-eval/src/ration_eval/suite.py packages/ration-eval/tests/test_suite.py
git commit -m "feat: generate code understanding tasks with exact ground truth

Four question kinds derived from the syntax tree: which file defines a
symbol, its parameters in order, which functions call it, and which module
an imported name came from. All answerable exactly, so no language model
grades anything and the numbers carry no judge variance.

Every task records the whole corpus as available and a small oracle set as
what is actually needed. That difference is the quantity phase 1 exists to
measure."
```

---

### Task 9: Graders

**Files:**
- Create: `packages/ration-eval/src/ration_eval/graders.py`
- Create: `packages/ration-eval/tests/test_graders.py`

**Interfaces:**
- Consumes: `Evaluator`, `TaskProfile`, `Outcome`, `Verdict`, `SuiteTask`
- Produces: `SuiteGrader(tasks: dict[str, SuiteTask])` with `grader_id = "exact-and-set-v1"`, implementing `Evaluator`. Quality is `1.0` or `0.0` for single answer kinds and the F1 of the answer set for `"callers"`.

- [ ] **Step 1: Write the failing test**

```python
# packages/ration-eval/tests/test_graders.py
from ration_core.contracts import FailureMode, Outcome, TaskProfile

from ration_eval.graders import SuiteGrader
from ration_eval.suite import SuiteTask


def _outcome(answer: str, failure=None) -> Outcome:
    return Outcome(
        answer=answer, tokens_in=1, tokens_out=1, context_tokens_materialised=1,
        wall_ms=1.0, executor_id="double", executor_version="1", failure_mode=failure,
    )


def _grader(kind: str, answer: set[str]) -> tuple[SuiteGrader, TaskProfile]:
    task = TaskProfile(task_id="t1", question="q", kind=kind)
    item = SuiteTask(task=task, answer=frozenset(answer), oracle_item_ids=("a",))
    return SuiteGrader({"t1": item}), task


def test_an_exact_answer_scores_one():
    grader, task = _grader("definition_file", {"pkg/maths.py"})
    assert grader.evaluate(task, _outcome("pkg/maths.py")).quality == 1.0


def test_case_and_surrounding_whitespace_do_not_change_the_score():
    grader, task = _grader("definition_file", {"pkg/maths.py"})
    assert grader.evaluate(task, _outcome("  PKG/Maths.py \n")).quality == 1.0


def test_a_wrong_answer_scores_zero():
    grader, task = _grader("definition_file", {"pkg/maths.py"})
    assert grader.evaluate(task, _outcome("pkg/app.py")).quality == 0.0


def test_an_answer_buried_in_a_sentence_still_counts():
    grader, task = _grader("definition_file", {"pkg/maths.py"})
    assert grader.evaluate(task, _outcome("It is defined in pkg/maths.py.")).quality == 1.0


def test_a_set_answer_is_scored_by_f1_not_all_or_nothing():
    grader, task = _grader("callers", {"double", "triple"})
    assert grader.evaluate(task, _outcome("double")).quality == 2 / 3


def test_a_perfect_set_answer_scores_one():
    grader, task = _grader("callers", {"double", "triple"})
    assert grader.evaluate(task, _outcome("double, triple")).quality == 1.0


def test_a_failed_run_scores_zero_and_records_the_diagnosis():
    grader, task = _grader("definition_file", {"pkg/maths.py"})
    verdict = grader.evaluate(task, _outcome("", FailureMode.EXECUTOR_ERROR))
    assert verdict.quality == 0.0
    assert "executor_error" in (verdict.failure_diagnosis or "")


def test_the_grader_identifies_itself_on_every_verdict():
    grader, task = _grader("definition_file", {"pkg/maths.py"})
    assert grader.evaluate(task, _outcome("x")).grader_id == "exact-and-set-v1"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest packages/ration-eval/tests/test_graders.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'ration_eval.graders'`

- [ ] **Step 3: Write minimal implementation**

```python
# packages/ration-eval/src/ration_eval/graders.py
"""Deterministic grading. No language model judges anything in phase 1."""

from __future__ import annotations

import re

from ration_core.contracts import Outcome, TaskProfile, Verdict

from ration_eval.suite import SuiteTask

SET_KINDS = {"callers"}


class SuiteGrader:
    grader_id = "exact-and-set-v1"

    def __init__(self, tasks: dict[str, SuiteTask]):
        self._tasks = dict(tasks)

    def evaluate(self, task: TaskProfile, outcome: Outcome) -> Verdict:
        expected = self._tasks[task.task_id].answer
        if outcome.failure_mode is not None:
            return Verdict(
                quality=0.0,
                grader_id=self.grader_id,
                failure_diagnosis=outcome.failure_mode.value,
            )

        got = outcome.answer.strip().lower()
        if task.kind in SET_KINDS:
            return Verdict(
                quality=_f1({e.lower() for e in expected}, set(_tokens(got))),
                grader_id=self.grader_id,
            )

        hit = any(e.lower() in got for e in expected)
        return Verdict(
            quality=1.0 if hit else 0.0,
            grader_id=self.grader_id,
            failure_diagnosis=None if hit else f"expected one of {sorted(expected)}",
        )


def _tokens(text: str) -> list[str]:
    return [t for t in re.split(r"[^\w.]+", text) if t]


def _f1(expected: set[str], got: set[str]) -> float:
    if not expected and not got:
        return 1.0
    matched = len({e for e in expected if e in got or e.split(".")[-1] in got})
    if matched == 0:
        return 0.0
    precision = matched / len(got) if got else 0.0
    recall = matched / len(expected) if expected else 0.0
    return 0.0 if precision + recall == 0 else 2 * precision * recall / (precision + recall)
```

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest packages/ration-eval/tests/test_graders.py -v`
Expected: PASS, 8 passed

- [ ] **Step 5: Commit**

```bash
git add packages/ration-eval/src/ration_eval/graders.py packages/ration-eval/tests/test_graders.py
git commit -m "feat: add deterministic graders for the code understanding suite

Substring matching for single answer kinds, so a model that answers in a
sentence is not penalised for politeness, and set F1 for the callers kind,
so a partially right answer is scored partially right rather than zero.

A failed run scores zero and carries the failure mode as its diagnosis,
so failures stay in the frontier instead of vanishing from the sample."
```

---

### Task 10: The four baselines

**Files:**
- Create: `packages/ration-eval/src/ration_eval/baselines.py`
- Create: `packages/ration-eval/tests/test_baselines.py`

**Interfaces:**
- Consumes: `Selector`, `TaskProfile`, `InformationStore`, `ContextItem`, `Resolution`, `SuiteTask`
- Produces: `FullContext()`, `FixedTruncation()`, `PlainRetrieval(k: int = 5)`, `Oracle(tasks: dict[str, SuiteTask])`, with `selector_id` values `"full-context"`, `"fixed-truncation"`, `"plain-retrieval"`, `"oracle"`. All four implement `Selector`. `bm25_scores(query, store, item_ids) -> dict[str, float]` is exported for reuse by Task 11.

- [ ] **Step 1: Write the failing test**

```python
# packages/ration-eval/tests/test_baselines.py
from pathlib import Path

import pytest
from ration_core.contracts import TaskProfile

from ration_eval.baselines import FixedTruncation, FullContext, Oracle, PlainRetrieval
from ration_eval.corpus import RepoCorpus
from ration_eval.suite import SuiteTask

FIXTURE = Path(__file__).parents[3] / "tests" / "fixtures" / "minirepo"


@pytest.fixture
def corpus():
    return RepoCorpus.from_path(FIXTURE)


def _task(corpus, question="Which file defines `add`?"):
    return TaskProfile(
        task_id="t1", question=question, kind="definition_file",
        available_item_ids=corpus.item_ids(),
    )


def _materialised(corpus, items):
    return sum(corpus.token_count(i.item_id, i.resolution) for i in items)


def test_every_baseline_is_a_selector_with_a_distinct_id(corpus):
    ids = {
        FullContext().selector_id,
        FixedTruncation().selector_id,
        PlainRetrieval().selector_id,
        Oracle({}).selector_id,
    }
    assert len(ids) == 4


def test_full_context_takes_everything_that_fits(corpus):
    items = FullContext().select(_task(corpus), corpus, token_budget=10_000)
    assert len(items) == len(corpus.item_ids())


def test_full_context_still_respects_the_budget(corpus):
    items = FullContext().select(_task(corpus), corpus, token_budget=5)
    assert _materialised(corpus, items) <= 5


def test_fixed_truncation_takes_a_prefix_in_corpus_order(corpus):
    items = FixedTruncation().select(_task(corpus), corpus, token_budget=10)
    assert [i.item_id for i in items] == list(corpus.item_ids())[: len(items)]


def test_plain_retrieval_ranks_the_relevant_item_first(corpus):
    items = PlainRetrieval(k=1).select(_task(corpus), corpus, token_budget=10_000)
    assert "add" in items[0].item_id


def test_the_oracle_selects_exactly_the_items_the_task_needs(corpus):
    task = _task(corpus)
    tasks = {
        "t1": SuiteTask(task=task, answer=frozenset({"pkg/maths.py"}),
                        oracle_item_ids=("pkg/maths.py::add",))
    }
    items = Oracle(tasks).select(task, corpus, token_budget=10_000)
    assert [i.item_id for i in items] == ["pkg/maths.py::add"]


def test_every_baseline_stays_within_budget_at_every_budget(corpus):
    tasks = {
        "t1": SuiteTask(task=_task(corpus), answer=frozenset({"x"}),
                        oracle_item_ids=("pkg/maths.py::add",))
    }
    for selector in (FullContext(), FixedTruncation(), PlainRetrieval(), Oracle(tasks)):
        for budget in (0, 1, 5, 50, 10_000):
            items = selector.select(_task(corpus), corpus, token_budget=budget)
            assert _materialised(corpus, items) <= budget, selector.selector_id
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest packages/ration-eval/tests/test_baselines.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'ration_eval.baselines'`

- [ ] **Step 3: Write minimal implementation**

```python
# packages/ration-eval/src/ration_eval/baselines.py
"""The four baselines.

They implement the same Selector port as the Ration policy, so a comparison
between them differs in exactly one object. That is what makes the comparison
in this project fair by construction rather than by care.
"""

from __future__ import annotations

import math
import re
from collections import Counter

from ration_core.contracts import ContextItem, Resolution, TaskProfile
from ration_core.ports import InformationStore

from ration_eval.suite import SuiteTask

K1 = 1.5
B = 0.75


def _fit(store: InformationStore, ordered_ids: list[str], budget: int) -> tuple[ContextItem, ...]:
    """Take items in the given order while they fit. Never exceed the budget."""
    chosen: list[ContextItem] = []
    used = 0
    for item_id in ordered_ids:
        cost = store.token_count(item_id, Resolution.RAW)
        if used + cost > budget:
            continue
        chosen.append(ContextItem(item_id=item_id, resolution=Resolution.RAW))
        used += cost
    return tuple(chosen)


class FullContext:
    """What everyone actually does. If Ration does not beat this there is no project."""

    selector_id = "full-context"

    def select(self, task, store, token_budget):
        return _fit(store, list(task.available_item_ids), token_budget)


class FixedTruncation:
    """Embarrassingly simple, and it beats sophisticated methods more often than admitted."""

    selector_id = "fixed-truncation"

    def select(self, task, store, token_budget):
        return _fit(store, sorted(task.available_item_ids), token_budget)


class PlainRetrieval:
    """Standard lexical top-k over the same information."""

    selector_id = "plain-retrieval"

    def __init__(self, k: int = 5):
        self._k = k

    def select(self, task, store, token_budget):
        scores = bm25_scores(task.question, store, task.available_item_ids)
        ranked = sorted(scores, key=lambda i: (-scores[i], i))[: self._k]
        return _fit(store, ranked, token_budget)


class Oracle:
    """An upper bound. Nothing can beat it. The distance to it is the headline."""

    selector_id = "oracle"

    def __init__(self, tasks: dict[str, SuiteTask]):
        self._tasks = dict(tasks)

    def select(self, task: TaskProfile, store, token_budget):
        needed = self._tasks[task.task_id].oracle_item_ids if task.task_id in self._tasks else ()
        return _fit(store, list(needed), token_budget)


def tokenize(text: str) -> list[str]:
    return [t.lower() for t in re.split(r"[^\w]+", text) if t]


def bm25_scores(
    query: str, store: InformationStore, item_ids: tuple[str, ...]
) -> dict[str, float]:
    """Okapi BM25 with the Lucene IDF variant, in pure Python.

    Written here rather than pulled in as a dependency so ration-eval stays
    installable with no wheels to build and the ranking is inspectable.
    """
    docs = {i: tokenize(store.get(i, Resolution.RAW) + " " + i) for i in item_ids}
    if not docs:
        return {}
    lengths = {i: len(t) for i, t in docs.items()}
    avg_len = sum(lengths.values()) / len(lengths)
    doc_freq: Counter[str] = Counter()
    for tokens in docs.values():
        doc_freq.update(set(tokens))

    n = len(docs)
    scores: dict[str, float] = {}
    query_terms = tokenize(query)
    for item_id, tokens in docs.items():
        counts = Counter(tokens)
        score = 0.0
        for term in query_terms:
            if term not in counts:
                continue
            df = doc_freq[term]
            idf = math.log(1 + (n - df + 0.5) / (df + 0.5))
            tf = counts[term]
            norm = tf + K1 * (1 - B + B * lengths[item_id] / avg_len)
            score += idf * (tf * (K1 + 1)) / norm
        scores[item_id] = score
    return scores
```

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest packages/ration-eval/tests/test_baselines.py -v`
Expected: PASS, 7 passed

- [ ] **Step 5: Commit**

```bash
git add packages/ration-eval/src/ration_eval/baselines.py packages/ration-eval/tests/test_baselines.py
git commit -m "feat: add the four baselines as Selector implementations

Full context, fixed truncation, plain lexical retrieval and the oracle.
They implement the same port as the Ration policy, so a comparison differs
in exactly one object rather than in a whole pipeline.

All four are budget respecting at every budget, verified across five
budgets including zero. A baseline that quietly exceeds its budget would
win the frontier by cheating, which is the most likely way this project
could fool itself.

BM25 is written out in pure Python rather than added as a dependency, so
the ranking is inspectable and there is nothing to build."
```

---

### Task 11: The Ration selector

**Files:**
- Create: `packages/ration-eval/src/ration_eval/selector.py`
- Create: `packages/ration-eval/tests/test_selector.py`

**Interfaces:**
- Consumes: `Selector`, `bm25_scores`, `Resolution`, `ContextItem`
- Produces: `BudgetedSelector(*, floor: float = 0.0, max_items: int | None = None)` with `selector_id = "ration-budgeted-v1"`.

The policy: rank by BM25 score divided by token cost, which is value per token rather than value alone, then take greedily while the budget allows, dropping anything scoring at or below `floor`. This is deliberately the simplest policy that is not a baseline. It exists to be beaten by phase 2, and the phase 1 result is meaningful whether it wins or loses.

- [ ] **Step 1: Write the failing test**

```python
# packages/ration-eval/tests/test_selector.py
from pathlib import Path

import pytest
from ration_core.contracts import Resolution, TaskProfile

from ration_eval.corpus import RepoCorpus
from ration_eval.selector import BudgetedSelector

FIXTURE = Path(__file__).parents[3] / "tests" / "fixtures" / "minirepo"


@pytest.fixture
def corpus():
    return RepoCorpus.from_path(FIXTURE)


def _task(corpus, question="Which file defines `add`?"):
    return TaskProfile(
        task_id="t1", question=question, kind="definition_file",
        available_item_ids=corpus.item_ids(),
    )


def _materialised(corpus, items):
    return sum(corpus.token_count(i.item_id, i.resolution) for i in items)


def test_it_never_exceeds_the_budget(corpus):
    for budget in (0, 1, 3, 12, 10_000):
        items = BudgetedSelector().select(_task(corpus), corpus, budget)
        assert _materialised(corpus, items) <= budget


def test_it_prefers_a_relevant_item_over_an_irrelevant_one(corpus):
    items = BudgetedSelector().select(_task(corpus), corpus, token_budget=12)
    assert any("add" in i.item_id for i in items)


def test_it_ranks_by_value_per_token_not_value_alone(corpus):
    class Store:
        def item_ids(self):
            return ("cheap", "expensive")

        def get(self, item_id, resolution):
            return "add" if item_id == "cheap" else "add " + "filler " * 200

        def token_count(self, item_id, resolution):
            return len(self.get(item_id, resolution).split())

    task = TaskProfile(
        task_id="t1", question="add", kind="definition_file",
        available_item_ids=("cheap", "expensive"),
    )
    items = BudgetedSelector().select(task, Store(), token_budget=300)
    assert items[0].item_id == "cheap"


def test_a_zero_scoring_item_is_never_selected(corpus):
    items = BudgetedSelector(floor=0.0).select(
        _task(corpus, question="zzzz nothing matches this"), corpus, token_budget=10_000
    )
    assert items == ()


def test_max_items_caps_the_selection(corpus):
    items = BudgetedSelector(max_items=1).select(_task(corpus), corpus, token_budget=10_000)
    assert len(items) <= 1


def test_selection_is_deterministic_for_the_same_inputs(corpus):
    first = BudgetedSelector().select(_task(corpus), corpus, 12)
    second = BudgetedSelector().select(_task(corpus), corpus, 12)
    assert first == second


def test_everything_it_selects_is_at_raw_resolution_in_phase_one(corpus):
    items = BudgetedSelector().select(_task(corpus), corpus, 10_000)
    assert all(i.resolution is Resolution.RAW for i in items)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest packages/ration-eval/tests/test_selector.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'ration_eval.selector'`

- [ ] **Step 3: Write minimal implementation**

```python
# packages/ration-eval/src/ration_eval/selector.py
"""The first Ration policy: minimum sufficient context under a token budget.

Deliberately the simplest policy that is not a baseline. It ranks by value per
token rather than by value, which is the only idea in it, and takes greedily
while the budget allows.

It exists to be beaten by phase 2. The phase 1 result is meaningful either way:
if it beats plain retrieval, value per token is worth something; if it does
not, that is reported and phase 2 starts from a known floor rather than an
assumed one.
"""

from __future__ import annotations

from ration_core.contracts import ContextItem, Resolution, TaskProfile
from ration_core.ports import InformationStore

from ration_eval.baselines import bm25_scores


class BudgetedSelector:
    selector_id = "ration-budgeted-v1"

    def __init__(self, *, floor: float = 0.0, max_items: int | None = None):
        self._floor = floor
        self._max_items = max_items

    def select(
        self, task: TaskProfile, store: InformationStore, token_budget: int
    ) -> tuple[ContextItem, ...]:
        scores = bm25_scores(task.question, store, task.available_item_ids)
        candidates = []
        for item_id, score in scores.items():
            if score <= self._floor:
                continue
            cost = store.token_count(item_id, Resolution.RAW)
            if cost == 0 or cost > token_budget:
                continue
            candidates.append((score / cost, score, item_id))

        candidates.sort(key=lambda c: (-c[0], -c[1], c[2]))

        chosen: list[ContextItem] = []
        used = 0
        for _density, _score, item_id in candidates:
            if self._max_items is not None and len(chosen) >= self._max_items:
                break
            cost = store.token_count(item_id, Resolution.RAW)
            if used + cost > token_budget:
                continue
            chosen.append(ContextItem(item_id=item_id, resolution=Resolution.RAW))
            used += cost
        return tuple(chosen)
```

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest packages/ration-eval/tests/test_selector.py -v`
Expected: PASS, 7 passed

- [ ] **Step 5: Commit**

```bash
git add packages/ration-eval/src/ration_eval/selector.py packages/ration-eval/tests/test_selector.py
git commit -m "feat: add the first Ration policy, minimum sufficient context

Ranks candidates by relevance per token rather than relevance, then takes
greedily while the budget allows. That single change is the whole policy,
and it is the simplest thing that is not already a baseline.

It exists to be beaten by phase 2. If it loses to plain retrieval, that is
reported and phase 2 starts from a measured floor rather than an assumed
one."
```

---

### Task 12: Metrics, the frontier and the oracle gap

**Files:**
- Create: `packages/ration-ledger/src/ration_ledger/report.py`
- Create: `packages/ration-ledger/tests/test_report.py`
- Modify: `packages/ration-ledger/pyproject.toml` (no change needed if already correct)

**Interfaces:**
- Consumes: `Record`, `SqliteLedger`, `ProfileMismatchError`
- Produces: `frontier(records) -> tuple[Point, ...]` where `Point` has `selector_id`, `budget`, `mean_quality`, `mean_materialised`, `mean_wall_ms`, `n`. `oracle_gap(points, *, oracle_id="oracle") -> dict[str, float]`. `render_markdown(ledger, *, suite_id, profile_id, selector_ids) -> str`.

- [ ] **Step 1: Write the failing test**

```python
# packages/ration-ledger/tests/test_report.py
import pytest
from ration_core.contracts import (
    HardwareProfile, Outcome, Plan, Record, TaskProfile, Verdict,
)

from ration_ledger.report import frontier, oracle_gap, render_markdown
from ration_ledger.store import SqliteLedger


def _record(run_id, selector, budget, quality, materialised, profile_id="p1"):
    return Record(
        run_id=run_id, seed=7, created_at="2026-09-22T00:00:00Z",
        profile=HardwareProfile(
            profile_id=profile_id, platform="test", memory_bytes=1,
            accelerator="none", measurable=("wall_ms",),
        ),
        task=TaskProfile(task_id=f"t-{run_id}", question="q", kind="definition_file"),
        plan=Plan(task_id=f"t-{run_id}", model="m", token_budget=budget, selector_id=selector),
        outcome=Outcome(
            answer="a", tokens_in=materialised, tokens_out=1,
            context_tokens_materialised=materialised, wall_ms=2.0,
            executor_id="double", executor_version="1",
        ),
        verdict=Verdict(quality=quality, grader_id="g"),
        suite_id="code-understanding", suite_version="v1", environment={},
    )


def test_a_frontier_point_is_one_selector_at_one_budget():
    points = frontier([
        _record("r1", "ration", 100, 1.0, 10),
        _record("r2", "ration", 100, 0.0, 20),
        _record("r3", "ration", 200, 1.0, 40),
    ])
    by_budget = {p.budget: p for p in points if p.selector_id == "ration"}
    assert by_budget[100].mean_quality == 0.5
    assert by_budget[100].mean_materialised == 15
    assert by_budget[100].n == 2
    assert by_budget[200].mean_quality == 1.0


def test_the_oracle_gap_is_the_distance_to_the_best_possible_selection():
    points = frontier([
        _record("r1", "oracle", 100, 1.0, 5),
        _record("r2", "ration", 100, 0.75, 20),
    ])
    assert oracle_gap(points)["ration"] == pytest.approx(0.25)


def test_the_oracle_has_no_gap_from_itself():
    points = frontier([_record("r1", "oracle", 100, 1.0, 5)])
    assert oracle_gap(points)["oracle"] == 0.0


def test_the_gap_is_not_computed_when_the_oracle_was_not_run():
    points = frontier([_record("r1", "ration", 100, 0.5, 5)])
    assert oracle_gap(points) == {}


def test_a_report_refuses_to_render_when_records_span_profiles(tmp_path):
    ledger = SqliteLedger(tmp_path / "l.db")
    ledger.write(_record("r1", "ration", 100, 1.0, 5, profile_id="p1"))
    ledger.write(_record("r2", "oracle", 100, 1.0, 5, profile_id="p2"))
    with pytest.raises(Exception):
        render_markdown(
            ledger, suite_id="code-understanding", profile_id=None,
            selector_ids=("ration", "oracle"),
        )


def test_a_rendered_report_names_the_profile_the_seed_and_the_suite_version(tmp_path):
    ledger = SqliteLedger(tmp_path / "l.db")
    ledger.write(_record("r1", "ration", 100, 1.0, 5))
    ledger.write(_record("r2", "oracle", 100, 1.0, 3))
    text = render_markdown(
        ledger, suite_id="code-understanding", profile_id="p1",
        selector_ids=("ration", "oracle"),
    )
    assert "p1" in text
    assert "seed 7" in text
    assert "v1" in text
    assert "oracle gap" in text.lower()


def test_a_selector_with_no_runs_is_named_in_the_report_not_omitted(tmp_path):
    ledger = SqliteLedger(tmp_path / "l.db")
    ledger.write(_record("r1", "ration", 100, 1.0, 5))
    text = render_markdown(
        ledger, suite_id="code-understanding", profile_id="p1",
        selector_ids=("ration", "full-context"),
    )
    assert "full-context" in text
    assert "no runs" in text
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest packages/ration-ledger/tests/test_report.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'ration_ledger.report'`

- [ ] **Step 3: Write minimal implementation**

```python
# packages/ration-ledger/src/ration_ledger/report.py
"""Report generation. Reads the ledger, and there is no other source of a number."""

from __future__ import annotations

from collections import defaultdict
from dataclasses import dataclass
from statistics import fmean

from ration_core.contracts import Record

from ration_ledger.store import SqliteLedger


@dataclass(frozen=True, slots=True)
class Point:
    selector_id: str
    budget: int
    mean_quality: float
    mean_materialised: float
    mean_wall_ms: float
    n: int


def frontier(records) -> tuple[Point, ...]:
    grouped: dict[tuple[str, int], list[Record]] = defaultdict(list)
    for record in records:
        grouped[(record.plan.selector_id, record.plan.token_budget)].append(record)

    points = [
        Point(
            selector_id=selector_id,
            budget=budget,
            mean_quality=fmean(r.verdict.quality for r in group),
            mean_materialised=fmean(r.outcome.context_tokens_materialised for r in group),
            mean_wall_ms=fmean(r.outcome.wall_ms for r in group),
            n=len(group),
        )
        for (selector_id, budget), group in grouped.items()
    ]
    return tuple(sorted(points, key=lambda p: (p.selector_id, p.budget)))


def oracle_gap(points, *, oracle_id: str = "oracle") -> dict[str, float]:
    """Distance from the best possible selection, per selector, at its best budget.

    Returns an empty mapping when the oracle was not run. A gap computed
    against a missing upper bound would be a number with no meaning.
    """
    best: dict[str, float] = {}
    for point in points:
        best[point.selector_id] = max(best.get(point.selector_id, 0.0), point.mean_quality)
    if oracle_id not in best:
        return {}
    ceiling = best[oracle_id]
    return {sid: ceiling - quality for sid, quality in sorted(best.items())}


def render_markdown(
    ledger: SqliteLedger, *, suite_id: str, profile_id: str | None, selector_ids: tuple[str, ...]
) -> str:
    groups = ledger.comparable(
        suite_id=suite_id, selector_ids=selector_ids, profile_id=profile_id
    )
    records = [r for group in groups.values() for r in group]
    if not records:
        return f"# {suite_id}\n\nNo runs recorded.\n"

    first = records[0]
    points = frontier(records)
    gaps = oracle_gap(points)

    lines = [
        f"# {suite_id} {first.suite_version}",
        "",
        "Generated from the ledger. Nothing in this file was written by hand.",
        "",
        f"- Hardware profile: `{first.profile.profile_id}` ({first.profile.platform})",
        f"- Model: `{first.plan.model}`",
        f"- Seed: seed {first.seed}",
        f"- Suite: `{suite_id}` {first.suite_version}",
        f"- Runs: {len(records)}",
        "",
        "Context tokens are counted by whitespace splitting, which is this "
        "repository's own unit and not a model tokenizer.",
        "",
        "## Frontier",
        "",
        "| Selector | Budget | Mean quality | Mean materialised | Mean wall ms | Runs |",
        "|---|---:|---:|---:|---:|---:|",
    ]
    for point in points:
        lines.append(
            f"| {point.selector_id} | {point.budget} | {point.mean_quality:.3f} "
            f"| {point.mean_materialised:.1f} | {point.mean_wall_ms:.1f} | {point.n} |"
        )

    for selector_id, group in sorted(groups.items()):
        if not group:
            lines.append(f"| {selector_id} | | | | | no runs |")

    lines += ["", "## Oracle gap", ""]
    if not gaps:
        lines.append("Not computed. The oracle selector was not run for this suite.")
    else:
        lines += ["| Selector | Gap from oracle |", "|---|---:|"]
        lines += [f"| {sid} | {gap:.3f} |" for sid, gap in gaps.items()]

    unmeasurable = [
        name
        for name in ("kv_bytes", "bytes_moved", "energy_j", "cost_usd")
        if not first.profile.can_measure(name)
    ]
    if unmeasurable:
        lines += [
            "",
            "## Not measurable on this profile",
            "",
            "Recorded as null rather than estimated, per ADR 0004: "
            + ", ".join(f"`{n}`" for n in unmeasurable)
            + ".",
        ]
    return "\n".join(lines) + "\n"
```

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest packages/ration-ledger -v`
Expected: PASS, 12 passed

- [ ] **Step 5: Commit**

```bash
git add packages/ration-ledger/src/ration_ledger/report.py packages/ration-ledger/tests/test_report.py
git commit -m "feat: generate the frontier and the oracle gap from the ledger

Quality and cost on the same axes, one point per selector per budget,
rather than a single scalar that would let any system look good by
choosing the weighting after seeing the results.

The oracle gap is empty rather than zero when the oracle was not run. A
gap measured against a missing upper bound is a number with no meaning.

Every report names its hardware profile, model, seed and suite version,
lists selectors that produced no runs instead of omitting them, and states
which fields this profile cannot measure."
```

---

### Task 13: The command line

**Files:**
- Create: `packages/ration-cli/pyproject.toml`
- Create: `packages/ration-cli/src/ration_cli/__init__.py`
- Create: `packages/ration-cli/src/ration_cli/environment.py`
- Create: `packages/ration-cli/src/ration_cli/main.py`
- Create: `packages/ration-cli/tests/test_cli.py`
- Modify: root `pyproject.toml` to add `ration` to `[project.scripts]` via the CLI package

**Interfaces:**
- Consumes: everything from Tasks 1 to 12
- Produces: `ration suite build --repo PATH --out FILE --seed N [--limit N]`, `ration run --suite FILE --repo PATH --selector NAME --budget N [--budget N ...] --executor {double,ollama} [--model NAME] --ledger FILE --seed N`, `ration report --ledger FILE --suite ID [--profile ID] --out FILE`. `capture_environment() -> dict[str, str]`.

- [ ] **Step 1: Write the failing test**

```python
# packages/ration-cli/tests/test_cli.py
import json
from pathlib import Path

from ration_cli.environment import capture_environment
from ration_cli.main import main

FIXTURE = Path(__file__).parents[3] / "tests" / "fixtures" / "minirepo"


def test_environment_capture_records_what_reproduction_needs():
    env = capture_environment()
    for key in ("python", "platform", "machine", "ration_core_version"):
        assert key in env
        assert env[key]


def test_suite_build_writes_a_reproducible_file(tmp_path):
    out = tmp_path / "suite.json"
    assert main(["suite", "build", "--repo", str(FIXTURE), "--out", str(out), "--seed", "7"]) == 0
    first = out.read_text()
    main(["suite", "build", "--repo", str(FIXTURE), "--out", str(out), "--seed", "7"])
    assert out.read_text() == first
    assert json.loads(first)["suite_id"] == "code-understanding"


def test_run_records_one_row_per_task_per_budget(tmp_path):
    suite = tmp_path / "suite.json"
    ledger = tmp_path / "l.db"
    main(["suite", "build", "--repo", str(FIXTURE), "--out", str(suite), "--seed", "7"])
    code = main([
        "run", "--suite", str(suite), "--repo", str(FIXTURE),
        "--selector", "oracle", "--budget", "50", "--budget", "500",
        "--executor", "double", "--ledger", str(ledger), "--seed", "7",
    ])
    assert code == 0

    from ration_ledger.store import SqliteLedger

    store = SqliteLedger(ledger)
    profile = store.profiles()[0]
    records = store.records(suite_id="code-understanding", profile_id=profile)
    tasks = json.loads(suite.read_text())["tasks"]
    assert len(records) == len(tasks) * 2


def test_the_double_executor_answers_from_the_suite_so_runs_need_no_model(tmp_path):
    suite = tmp_path / "suite.json"
    ledger = tmp_path / "l.db"
    main(["suite", "build", "--repo", str(FIXTURE), "--out", str(suite), "--seed", "7"])
    main([
        "run", "--suite", str(suite), "--repo", str(FIXTURE), "--selector", "oracle",
        "--budget", "500", "--executor", "double", "--ledger", str(ledger), "--seed", "7",
    ])
    from ration_ledger.store import SqliteLedger

    store = SqliteLedger(ledger)
    records = store.records(
        suite_id="code-understanding", profile_id=store.profiles()[0]
    )
    assert all(r.verdict.quality == 1.0 for r in records)


def test_report_writes_markdown_generated_from_the_ledger(tmp_path):
    suite = tmp_path / "suite.json"
    ledger = tmp_path / "l.db"
    out = tmp_path / "report.md"
    main(["suite", "build", "--repo", str(FIXTURE), "--out", str(suite), "--seed", "7"])
    for selector in ("oracle", "ration", "full-context", "fixed-truncation", "plain-retrieval"):
        main([
            "run", "--suite", str(suite), "--repo", str(FIXTURE), "--selector", selector,
            "--budget", "50", "--executor", "double", "--ledger", str(ledger), "--seed", "7",
        ])
    assert main([
        "report", "--ledger", str(ledger), "--suite", "code-understanding", "--out", str(out)
    ]) == 0
    text = out.read_text()
    assert "Oracle gap" in text
    assert "ration-budgeted-v1" in text


def test_an_unknown_selector_is_rejected_with_the_valid_names(tmp_path):
    suite = tmp_path / "suite.json"
    main(["suite", "build", "--repo", str(FIXTURE), "--out", str(suite), "--seed", "7"])
    assert main([
        "run", "--suite", str(suite), "--repo", str(FIXTURE), "--selector", "nope",
        "--budget", "50", "--executor", "double", "--ledger", str(tmp_path / "l.db"),
        "--seed", "7",
    ]) != 0
```

- [ ] **Step 2: Run test to verify it fails**

Run: `uv run pytest packages/ration-cli -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'ration_cli'`

- [ ] **Step 3: Write minimal implementation**

```python
# packages/ration-cli/src/ration_cli/environment.py
from __future__ import annotations

import importlib.metadata
import platform
import sys


def capture_environment() -> dict[str, str]:
    """Everything a reader needs to reproduce a run, captured at run time."""
    env = {
        "python": sys.version.split()[0],
        "platform": platform.platform(),
        "machine": platform.machine(),
    }
    for package in ("ration-core", "ration-exec", "ration-eval", "ration-ledger"):
        key = package.replace("-", "_") + "_version"
        try:
            env[key] = importlib.metadata.version(package)
        except importlib.metadata.PackageNotFoundError:
            env[key] = "0.0.0+local"
    return env
```

```python
# packages/ration-cli/src/ration_cli/main.py
from __future__ import annotations

import argparse
import json
import platform
import sys
import uuid
from datetime import UTC, datetime
from pathlib import Path

from ration_core.contracts import HardwareProfile
from ration_core.kernel import run_once
from ration_eval.baselines import FixedTruncation, FullContext, Oracle, PlainRetrieval
from ration_eval.corpus import RepoCorpus
from ration_eval.graders import SuiteGrader
from ration_eval.selector import BudgetedSelector
from ration_eval.suite import Suite, build_suite
from ration_exec.double import DeterministicExecutor
from ration_exec.meter_apple import AppleMeter
from ration_exec.ollama import OllamaExecutor
from ration_ledger.report import render_markdown
from ration_ledger.store import SqliteLedger

from ration_cli.environment import capture_environment

SELECTORS = ("ration", "oracle", "full-context", "fixed-truncation", "plain-retrieval")


def _profile() -> HardwareProfile:
    if sys.platform == "darwin":
        return AppleMeter().profile()
    return HardwareProfile(
        profile_id=f"generic-{platform.machine()}",
        platform=platform.platform(),
        memory_bytes=0,
        accelerator="unknown",
        measurable=("tokens_in", "tokens_out", "context_tokens_materialised", "wall_ms"),
    )


def _selector(name: str, tasks: dict):
    match name:
        case "ration":
            return BudgetedSelector()
        case "oracle":
            return Oracle(tasks)
        case "full-context":
            return FullContext()
        case "fixed-truncation":
            return FixedTruncation()
        case "plain-retrieval":
            return PlainRetrieval()
    raise SystemExit(f"unknown selector {name!r}. Valid names: {', '.join(SELECTORS)}")


def _cmd_suite_build(args) -> int:
    corpus = RepoCorpus.from_path(Path(args.repo))
    suite = build_suite(corpus, seed=args.seed, limit=args.limit)
    Path(args.out).write_text(json.dumps(suite.to_dict(), indent=2, sort_keys=True) + "\n")
    print(f"wrote {len(suite.tasks)} tasks to {args.out}")
    return 0


def _cmd_run(args) -> int:
    suite = Suite.from_dict(json.loads(Path(args.suite).read_text()))
    corpus = RepoCorpus.from_path(Path(args.repo))
    by_id = {t.task.task_id: t for t in suite.tasks}
    selector = _selector(args.selector, by_id)
    grader = SuiteGrader(by_id)
    ledger = SqliteLedger(args.ledger)
    profile = _profile()
    environment = capture_environment()

    if args.executor == "double":
        executor = DeterministicExecutor(
            answers={tid: sorted(t.answer)[0] for tid, t in by_id.items()}
        )
        model = "double:v1"
    else:
        executor = OllamaExecutor(args.model, seed=args.seed)
        model = f"ollama:{args.model}"

    written = 0
    for budget in args.budget:
        for item in suite.tasks:
            record = run_once(
                task=item.task,
                store=corpus,
                selector=selector,
                executor=executor,
                evaluator=grader,
                model=model,
                token_budget=budget,
                profile=profile,
                seed=args.seed,
                suite_id=suite.suite_id,
                suite_version=suite.suite_version,
                environment=environment,
                run_id=str(uuid.uuid4()),
                now=lambda: datetime.now(UTC).isoformat(),
            )
            ledger.write(record)
            written += 1
    print(f"recorded {written} runs for selector {selector.selector_id}")
    return 0


def _cmd_report(args) -> int:
    ledger = SqliteLedger(args.ledger)
    text = render_markdown(
        ledger,
        suite_id=args.suite,
        profile_id=args.profile,
        selector_ids=(
            "ration-budgeted-v1", "oracle", "full-context",
            "fixed-truncation", "plain-retrieval",
        ),
    )
    Path(args.out).write_text(text)
    print(f"wrote {args.out}")
    return 0


def main(argv: list[str] | None = None) -> int:
    parser = argparse.ArgumentParser(prog="ration")
    sub = parser.add_subparsers(dest="command", required=True)

    suite_parser = sub.add_parser("suite").add_subparsers(dest="suite_command", required=True)
    build = suite_parser.add_parser("build")
    build.add_argument("--repo", required=True)
    build.add_argument("--out", required=True)
    build.add_argument("--seed", type=int, required=True)
    build.add_argument("--limit", type=int, default=None)
    build.set_defaults(func=_cmd_suite_build)

    run = sub.add_parser("run")
    run.add_argument("--suite", required=True)
    run.add_argument("--repo", required=True)
    run.add_argument("--selector", required=True, choices=SELECTORS)
    run.add_argument("--budget", type=int, action="append", required=True)
    run.add_argument("--executor", choices=("double", "ollama"), default="double")
    run.add_argument("--model", default="qwen3:4b")
    run.add_argument("--ledger", required=True)
    run.add_argument("--seed", type=int, required=True)
    run.set_defaults(func=_cmd_run)

    report = sub.add_parser("report")
    report.add_argument("--ledger", required=True)
    report.add_argument("--suite", required=True)
    report.add_argument("--profile", default=None)
    report.add_argument("--out", required=True)
    report.set_defaults(func=_cmd_report)

    try:
        args = parser.parse_args(argv)
        return args.func(args)
    except SystemExit as exc:
        return exc.code if isinstance(exc.code, int) else 2


if __name__ == "__main__":
    raise SystemExit(main())
```

```toml
# packages/ration-cli/pyproject.toml
[project]
name = "ration-cli"
version = "0.0.0"
description = "The ration command line."
requires-python = ">=3.13"
dependencies = ["ration-core", "ration-exec", "ration-eval", "ration-ledger"]

[project.scripts]
ration = "ration_cli.main:main"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/ration_cli"]

[tool.uv.sources]
ration-core = { workspace = true }
ration-exec = { workspace = true }
ration-eval = { workspace = true }
ration-ledger = { workspace = true }
```

**Note for the implementer:** `_cmd_run` currently loses the question when using the Ollama executor, because `Plan` carries `task_id` and not the question. Apply the `Plan.question` change noted in Task 5 and set it in `kernel.run_once` from `task.question`, with a kernel test asserting `record.plan.question == task.question`.

- [ ] **Step 4: Run test to verify it passes**

Run: `uv run pytest packages/ration-cli -v`
Expected: PASS, 6 passed

- [ ] **Step 5: Run the whole suite and the import contract**

Run: `uv run pytest && uv run lint-imports && uv run ruff check packages/ && uv run ruff format --check packages/`
Expected: all pass, `Contracts: 2 kept, 0 broken.`

- [ ] **Step 6: Commit**

```bash
git add packages/ration-cli
git commit -m "feat: add the ration command line

Three commands: suite build, run and report. A suite built twice with the
same seed is byte identical, which is asserted rather than assumed.

Every run captures the Python version, platform, machine and package
versions into the record, so a result can be reproduced from the ledger
alone rather than from a memory of how it was produced.

The double executor answers from the suite's own ground truth, so the full
pipeline is exercisable end to end with no model and no network."
```

---

### Task 14: The phase 1 measurement, and the exit criterion

**Files:**
- Create: `docs/benchmarks/phase-1.md` (generated, never hand edited)
- Create: `docs/concepts/minimum-sufficient-context.md`
- Modify: `ROADMAP.md` (tick the phase 1 boxes that are genuinely done)
- Modify: `CHANGELOG.md`

**Interfaces:**
- Consumes: the `ration` command line
- Produces: the first measured result in the repository

- [ ] **Step 1: Choose and pin the corpus repository**

Use a repository you did not write, so the model cannot have an unfair familiarity advantage, and pin it by commit. Record the choice and the SHA in `docs/benchmarks/phase-1.md` by passing them through the suite file.

```bash
git clone --depth 50 https://github.com/psf/requests /tmp/corpus-requests
cd /tmp/corpus-requests && git rev-parse HEAD   # record this SHA
```

- [ ] **Step 2: Build the suite**

```bash
uv run ration suite build --repo /tmp/corpus-requests --out suites/code-understanding-v1.json --seed 7
```

Expected: a task count is printed. If it is under 200, raise `--limit` or pick a larger corpus. Under 200 tasks the means in the report will be too noisy to support any claim.

- [ ] **Step 3: Verify the pipeline end to end with the double, before spending model time**

```bash
uv run ration run --suite suites/code-understanding-v1.json --repo /tmp/corpus-requests \
  --selector oracle --budget 400 --executor double --ledger .ration/phase1.db --seed 7
```

Expected: quality 1.0 for every run. If it is not 1.0, the grader and the suite disagree and that is a bug to fix before any model runs. Nothing measured after this point is trustworthy until this passes.

- [ ] **Step 4: Run all five selectors across the budget sweep with a real model**

```bash
ollama pull qwen3:4b
for s in oracle ration full-context fixed-truncation plain-retrieval; do
  uv run ration run --suite suites/code-understanding-v1.json --repo /tmp/corpus-requests \
    --selector "$s" --budget 100 --budget 200 --budget 400 --budget 800 --budget 1600 \
    --executor ollama --model qwen3:4b --ledger .ration/phase1.db --seed 7
done
```

- [ ] **Step 5: Generate the report**

```bash
uv run ration report --ledger .ration/phase1.db --suite code-understanding --out docs/benchmarks/phase-1.md
```

- [ ] **Step 6: Check the exit criterion honestly**

The phase 1 exit criterion is: a frontier against all four baselines on the primary machine, reproducible from a seed, with the oracle gap reported, and the run reconstructable from the ledger alone.

Verify each one, and do not tick a box that is not true.

```bash
test -s docs/benchmarks/phase-1.md && grep -q "Oracle gap" docs/benchmarks/phase-1.md && echo "frontier and gap present"
grep -q "full-context" docs/benchmarks/phase-1.md && grep -q "fixed-truncation" docs/benchmarks/phase-1.md \
  && grep -q "plain-retrieval" docs/benchmarks/phase-1.md && grep -q "oracle" docs/benchmarks/phase-1.md \
  && echo "all four baselines present"
uv run python -c "
from ration_ledger.store import SqliteLedger
s = SqliteLedger('.ration/phase1.db')
p = s.profiles()[0]
r = s.records(suite_id='code-understanding', profile_id=p)[0]
assert r.seed and r.environment and r.profile.profile_id and r.suite_version
print('a record alone carries seed, environment, profile and suite version')
"
```

- [ ] **Step 7: Write the concept note the result needs**

`docs/concepts/minimum-sufficient-context.md` must state, in prose a reader can check against the report: what the token unit is and that it is not a model tokenizer, how the oracle is constructed for each of the four question kinds, why the oracle gap rather than absolute quality is the headline, and what this suite cannot tell you. That last section is required, not optional. At minimum it says that four syntactic question kinds over one Python repository is narrow evidence, and that a result here does not transfer to multi hop reasoning or to long conversation without being measured there.

- [ ] **Step 8: Record the outcome, including if the Ration policy lost**

If `ration-budgeted-v1` did not beat `plain-retrieval`, say so in `docs/benchmarks/phase-1.md` under a heading that says it plainly, and keep the numbers. That is a legitimate phase 1 result: it means value per token is not enough on its own, and phase 2 starts from a measured floor rather than an assumed one. Do not tune the selector until it wins and then report the tuned version as the phase 1 result. If you do tune it, that is a new selector id and a new set of runs.

- [ ] **Step 9: Commit**

```bash
git add docs/benchmarks/phase-1.md docs/concepts/minimum-sufficient-context.md \
        suites/code-understanding-v1.json ROADMAP.md CHANGELOG.md
git commit -m "feat: record the phase 1 measurement

First measured result in this repository. Frontier across five selectors
and five budgets on the code understanding suite, generated from the ledger
by ration report and not written by hand.

The concept note states the token unit, how the oracle is built for each
question kind, and what this suite cannot tell you. Four syntactic question
kinds over one repository is narrow evidence and the note says so."
```

---

## Self review

Run against the spec on 2026-09-22.

**Spec coverage.** Every phase 1 line in `ROADMAP.md` maps to a task: contracts to Task 1, ports and the core independence rule to Task 2, the kernel to Task 3, executor adapters and the meter to Tasks 4 and 5, the ledger and its cross profile refusal to Task 6, the evaluator to Task 9, the four baselines to Task 10, the selector to Task 11, the three commands to Task 13, reproducibility to Tasks 13 and 14, and the measured frontier to Task 14. The information store is Task 7 and the task suite is Task 8, both of which the roadmap implied through the exit criterion rather than listing, and both are now explicit.

**Open questions from the design specification.** Two of the five are closed by this plan: the task suite is fixed as code understanding over a repository with the reason recorded in Task 7, and the oracle construction is fixed per question kind in Task 8. Two remain genuinely open and are not touched here because they belong to later phases: the sufficiency signal is phase 3, and the Loomrun roadmap amendment is owed to that repository. The fifth, whether energy is measurable on Apple silicon, is answered in Task 4 by declaring it unmeasurable and recording null, which is the honest answer rather than a deferral.

**Placeholder scan.** No TBD, no "add error handling", no "similar to Task N". Three tasks carry an explicit "Note for the implementer" describing a known defect in the code shown and requiring it be fixed before commit: the `Plan.question` field in Tasks 5 and 13, and self recursive calls in the caller index in Task 8. These are deliberate. The code in a plan that has not been run is not to be trusted, and naming the specific places it is wrong is more useful than implying it is all correct.

**Type consistency.** `selector_id` is the attribute name on every selector and the column name in the ledger. The Ration policy's id is `ration-budgeted-v1` everywhere, including the CLI's `--selector ration` flag mapping and the report's selector list. `count_tokens` lives in `ration_exec.double` and is imported from there by `ration_exec.ollama`. `Resolution.RAW` is the only resolution phase 1 stores, and `RepoCorpus.get` raises on any other, so a phase 2 selector asking for a summary fails loudly here rather than silently receiving raw text.

**Known simplification to revisit in phase 2.** `bm25_scores` rebuilds the index on every call, which is O(corpus) per task. At phase 1 scale that is acceptable and it keeps the ranking inspectable. It will not survive phase 2 and should be replaced with a built index at that point, not before, because premature caching here would hide a correctness bug behind a performance layer.
