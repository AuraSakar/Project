# AGENTS.md

> Operational instructions for AI coding agents (Muse, ChatGPT, Claude, Cursor, or any other tool) working in this repository. Follow this file; direct user/task instructions take precedence on conflict.

## 1. Project identity

SatQuery AI (SIH 26167) is an evidence-grounded, sensor-aware remote-sensing analysis system with a natural-language interface. Core rule: **the language model may explain evidence; it must not manufacture it.** Numeric/spatial claims must come from registered deterministic/model/GIS outputs, never visual guessing. Details: `PROJECT_OVERVIEW.md`; current state: `CONTEXT.md`.

## 2. Sources of truth (in order)

1. Task-relevant code under `satquery/`, `apps/api/`, `ml/` — read it before editing.
2. `models/registry.yaml`, `satquery/registry/tools.yaml`, `satquery/registry/preprocessing.yaml`, `experiments/phase5_backend/backend_contract.yaml` — frozen contracts.
3. `docs/REQUIREMENTS.md`, `docs/ARCHITECTURE.md`, `docs/EVALUATION.md`, `docs/FAILURE_POLICY.md` — policy.
4. `docs/DEVELOPMENT_PLAN.md` — phase status (volatile; never put status here).
5. `tests/` — executable specification. If prose and code disagree, report it; do not silently pick one.

## 3. Architecture boundaries

Flow: interpreter → feasibility → planner → registry → execution engine → adapters → inference/analytics → evidence → verification → composed answer. Transport (`apps/api`), science (`satquery`, `ml`), benchmarks, and orchestration stay separated. Never introduce arbitrary Python/shell/SQL, filesystem paths, URLs, model IDs, or tool IDs through the agent path. Planner may select only registered tools/models; execution must go through registered adapters.

## 4. Registry-first

Tools, models, preprocessing profiles, and capabilities exist only via registration: a YAML declaration **plus** a code-owned executor/implementation allow-list (see `satquery/registry/tools.py`, `satquery/registry/models.py`). Adding a capability means: executor enum → implementation map → `tools.yaml` entry (bounded parameter types only) → capability document → planner policy → adapter → tests. Never bypass the registry with a generic fallback.

## 5. Pipeline stages (do not conflate)

- **Interpreter** (`satquery/agent/interpreter.py`): deterministic lexical rules → typed `QueryIntent`. Selects data intent, never a tool.
- **Feasibility** (`satquery/agent/feasibility.py`): fail-closed gates → `ALLOW / ALLOW_WITH_WARNING / REQUEST_INPUT / ABSTAIN / REJECT`.
- **Planner** (`satquery/agent/planner.py`): bounded `ExecutionPlan` from server-issued IDs only. No query text or executable content in `PlanStep`.
- **Execution** (`satquery/execution/engine.py`, `jobs.py`): topological run, durable jobs/analyses, bounded queue, cancellation, optional exact-key result cache.
- **Adapters** (`satquery/execution/adapters.py`): only bridge from `ToolCall` + `ExecutionContext` to science code. Question text travels via job metadata, never as a free-form plan parameter; never put live service objects into persisted metadata.
- **Inference/analytics**: frozen checkpoints (`satquery/inference/vqa.py`) and deterministic GIS (`satquery/analytics/`).

## 6. Evidence and provenance

Every answer must trace to model output, spatial evidence, raster statistics, metadata, or deterministic measurement. Preserve sensor/modality, CRS/affine/bounds, band roles, radiometric domain, and hashes. VQA text never creates masks, measurements, or confidence. Keep `operation_id`, model revision/SHA, preprocessing version, and parent-evidence links intact.

## 7. Frozen artifacts (do not rewrite history)

`experiments/` closeouts, manifests, and diagnostics are evidence, not scratch space. Never regenerate or edit them merely to make tests agree, and never manufacture a `PASS`/metric/prediction. If a committed artifact disagrees with its generator, verify which side is authoritative (primary source wins over derived files), reproduce with the documented generator command, confirm the diff is minimal and scientifically neutral, and report it. Sealed test data must never guide tuning.

## 8. Testing discipline

- Before editing: `git status`, `git log --oneline -10`; run relevant focused tests.
- During: `python -m pytest tests/<area>/... -q` for the touched area.
- Before declaring success: broader affected suite, then full `python -m pytest -q`, plus `git diff --check`.
- Tests must prove behavior (deterministic/geospatial/contract tests first), not string-match source. Do not inflate counts with trivial assertions; report skipped/unrunnable checks explicitly. Current observed baseline is 733 passed — re-verify, never assume.

## 9. Minimal change

Smallest registry-consistent fix wins. Reuse existing patterns, keep transport/science/benchmark layers separate, avoid unrelated refactors and speculative infrastructure (no new K8s/Kafka/vector-DB/multi-agent scaffolding without a demonstrated requirement).

## 10. Dependencies and models

`pyproject.toml` is the dependency source. Do not change versions casually; never reinstall PyTorch speculatively. Prefer declared extras: `.[dev]`, `.[training]`, `.[grounding]`, `.[multisensor]` — use the extra instead of hand-adding packages. Never change model IDs, revisions, checkpoint files/SHAs, or preprocessing contracts to “fix” a failure; pin revisions and verify hashes.

## 11. Data, security, git

Treat rasters/archives as untrusted: validate sizes, dimensions, bands, paths; prevent traversal/unsafe extraction. Never commit credentials, `.env` contents, private URLs, datasets, or weights; keep `data/` and `models/cache/` out of Git. Network access stays opt-in (`ENABLE_REMOTE_NETWORK=false` locally). Never execute arbitrary code from queries. Inspect `git status` before/after; no force-push, history rewrites, or amending others’ commits; no `git commit`/`push`/PR unless explicitly requested.

## 12. Docs and frontend boundary

Docs must describe the actual implementation; label planned work as planned. Backend v1 contracts are stable — frontend work (`apps/web`) consumes `POST /api/v1/observations`, `POST /api/v1/query`, `GET /api/v1/jobs/{id}`, `GET /api/v1/analyses/{id}` (+ evidence/trace/artifacts) and display/tiles; it must not mint IDs, measurements, or completion states locally.

## 13. Debugging sequence

Inspect → reproduce → isolate root cause → smallest fix → focused tests → affected suite → full suite → `git diff --check` → report what changed, what was verified, and residual risks. Distinguish implemented → locally verified → externally evaluated → supported.

## 14. DO NOT

- Bypass planner/registry/adapters; add generic tool fallbacks; embed free-text questions in `PlanStep`.
- Treat SAR as RGB; assume sensor/polarization/radiometric units; compare pixels without a verified grid.
- Derive area/counts from prose; cache question-dependent results by plan alone.
- Load the same checkpoint twice via competing service instances; use unsafe globals for services.
- Download full benchmarks or retrain just to “see”; tune after seeing sealed-test results.
- Claim a test/benchmark/download/deployment ran unless it did.
