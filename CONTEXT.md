# CONTEXT.md — agent handoff (read first, 5–10 min)

> Living handoff. Volatile status lives here, not in `AGENTS.md`. See `PROJECT_OVERVIEW.md` for the full architecture. Note: a lowercase `context.md` may exist from earlier work; this file is the maintained handoff.

## Current milestone

Backend is stable: **733 passed, 3 warnings** observed (re-verify with `python -m pytest -q`). Canonical `SINGLE_VQA` works end-to-end. Phase-2C diagnostic, Phase-3A VRSBench images, and Phase-4 closeout-hash issues are resolved. No commit/push performed.

## Architecture in 60 seconds

`DeterministicQueryInterpreter` → `FeasibilityValidator` → `BoundedPlanner` (registry-only) → `ExecutionEngine`/`JobRunner` → registered adapters (`satquery/execution/adapters.py`) → science (`satquery/inference/vqa.py`, `satquery/analytics/`) → typed evidence → `AnalysisVerifier` → `AnswerComposer`. Transport lives in `apps/api/`; science in `satquery/` + `ml/`. Query text never enters `PlanStep`; the VQA question travels via job metadata.

## What works now

- Ingestion: `POST /api/v1/observations` → immutable GeoTIFF + visualization + tiles.
- VQA: `POST /api/v1/query/plan` (dry-run) and `POST /api/v1/query` (`202` → `analysis_id`/`job_id`) → `GET /api/v1/jobs/{id}` / `GET /api/v1/analyses/{id}` (+ `/evidence`, `/trace`, `/artifacts`) → `VqaEvidence` from pinned SmolVLM.
- Deterministic tools registered: `sar_temporal_change_v1`, `compute_mask_area_v1`, `mask_agreement_v1`, `single_image_vqa_v1` (`GET /api/v1/tools`, `/api/v1/system/capabilities`).
- Legacy `/api/vqa`, `/api/observations`, `/tiles` kept for compatibility.

## Pinned VQA facts (do not change)

- Registry ID `smolvlm_256m_instruct_v1` = `HuggingFaceTB/SmolVLM-256M-Instruct`, rev `7e3e67edbbed1bf9888184d9df282b700a323964`, `model.safetensors` SHA `74dea5904032e5ae99a2e0eef5179e6ac0f1dedc3ab0c7c2a5d4d387c843203e`, preprocessing `smolvlm_single_image_v1`. CPU default, `ENABLE_REMOTE_NETWORK=false`, SHA verified before load, single shared service instance, VQA never cached by plan alone.

## Key files

- Agent: `satquery/agent/{interpreter,feasibility,planner,composer}.py`
- Registry: `models/registry.yaml`, `satquery/registry/{tools.yaml,preprocessing.yaml,tools.py,models.py}`, `experiments/phase5_backend/backend_contract.yaml`
- Execution: `satquery/execution/{engine,jobs,adapters}.py`, `apps/api/app/{main.py,routes/v1_query.py}`
- Evidence: `satquery/evidence/{models,graph}.py`, `satquery/verification/analysis.py`
- Policy: `docs/{REQUIREMENTS,ARCHITECTURE,EVALUATION,FAILURE_POLICY,DEVELOPMENT_PLAN}.md`
- Frontend scaffold: `apps/web/` (React+Vite; consumes v1 API; no `dist` build — planned)

## Priorities / debt

1. Frontend completion against stable v1 contracts (planned).
2. Blocked specialists (ChangerEx/Chg2Cap/STURM) stay `BLOCKED` until dataset/checkpoint contracts unblock — no timeline.
3. `NOT_IMPLEMENTED`: spectral-index tools, generic temporal difference, auto-reprojection, PDF reports.
4. Keep `experiments/` provenance byte-accurate; `data/` + `models/cache/` stay git-ignored.

## Safe workflow

`git status` → read task-relevant sources → smallest registry-consistent fix → focused tests → full suite → `git diff --check` → report. Never bypass registry/planner/adapters; never put secrets/datasets/weights in Git; no commit/push unless asked.

## Do not break

Working `SINGLE_VQA`, registry allow-lists, SHA/revision pins, evidence schemas, v1 error envelopes/operation IDs, frozen `experiments/` history.
