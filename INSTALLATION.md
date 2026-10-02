# INSTALLATION.md — from clean Linux to running backend

> Two tracks: **Quick start** (run the app) vs **full development/experiment setup** (benchmarks, training). Do not download every benchmark unless you need it. Never commit secrets, datasets, or weights.

## 1. Prerequisites

- Linux with Python 3.11+ (verified with 3.12.3: `python3 --version`), `git`, and a C toolchain sufficient for `rasterio` wheels.
- Node 18+ + `npm` only if you touch `apps/web/`.
- No GPU required: the frozen VQA backend runs on CPU by default.

## 2. Clone

```bash
git clone <repo-url>
cd SIH-26167-SATQuery
git status
```

## 3. QUICK START (run the app)

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -e ".[dev]"
cp .env.example .env   # optional local overrides; leave SATQUERY_API_KEY blank locally
uvicorn apps.api.app.main:app --reload
```

- API: `http://127.0.0.1:8000`
- Swagger: `http://127.0.0.1:8000/docs`
- Health: `/health/live`, `/health/ready`

Frontend dev (separate terminal, optional):

```bash
cd apps/web
npm ci
npm run dev
```

Vite proxies `/api`, `/health`, `/tiles` to `http://127.0.0.1:8000` by default.

## 4. Verify

```bash
python -m pytest tests/registry/test_tools.py -q
python -m pytest -q
git diff --check
```

Observed baseline: **733 passed, 3 warnings** (snapshot, not a guarantee).

Smoke test (needs one ingested observation; IDs are server-issued):

```bash
curl -s -X POST http://127.0.0.1:8000/api/v1/query/plan \
  -H 'Content-Type: application/json' \
  -d '{"query":"What is visible in this image?","observation_ids":["obs_<id>"]}'
```

## 5. Environment variables (`.env.example` is authoritative)

| Variable | Default | Notes |
|---|---|---|
| `DATA_ROOT` / `MODEL_ROOT` | `./data` / `./models` | Local roots; both git-ignored for large assets |
| `PORT` | `8000` | Dev server port |
| `SATQUERY_API_KEY` | blank (auth off) | Set a secret only in deployment, never commit it |
| `SATQUERY_JOB_WORKERS`, `SATQUERY_TOOL_TIMEOUT_SECONDS`, `SATQUERY_MAX_QUEUED_JOBS` | `1` / `300` / `32` | Bounded local queue |
| `GPU_DEVICE` | `cpu` | VQA device; non-CPU requires CUDA |
| `ENABLE_REMOTE_NETWORK` | `false` | Keep `false` locally; models must be cached |
| `VQA_CPU_THREADS`, `GROUNDING_CPU_THREADS` | `2` | Runtime thread budgets |

Raster/visualization ceilings (`MAX_UPLOAD_SIZE_MB`, `MAX_RASTER_*`, `MAX_TILE_ZOOM`, …) are also in `.env.example`.

## 6. FULL DEVELOPMENT SETUP

```bash
pip install -e ".[training]"     # offline adaptation/eval (accelerate, peft, …)
pip install -e ".[grounding]"     # VRSBench materialization (fsspec[http], …)
pip install -e ".[multisensor]"   # multisensor tooling (pyarrow, zstandard, …)
```

Use the extra; do not hand-add equivalent packages. Never reinstall PyTorch speculatively.

## 7. Models (pinned, SHA-verified)

- Registry: `models/registry.yaml`; preprocessing: `satquery/registry/preprocessing.yaml`.
- Canonical VQA: `smolvlm_256m_instruct_v1` (`HuggingFaceTB/SmolVLM-256M-Instruct`, rev `7e3e67edbbed1bf9888184d9df282b700a323964`, `model.safetensors` SHA `74dea5904032e5ae99a2e0eef5179e6ac0f1dedc3ab0c7c2a5d4d387c843203e`). Resolves under `MODEL_ROOT` (Hugging Face cache layout) and is hash-checked before load. Never change IDs/revisions/SHAs to fix a failure.
- GroundingDINO / BIFOLD entries exist; their local runtimes are not probed here. Blocked specialists (ChangerEx/Chg2Cap/STURM) stay unrunnable until contracts unblock.

## 8. Data / benchmarks (only what you need)

- App runtime needs only `DATA_ROOT` (ingested observations + SQLite). Nothing to pre-download.
- Smoke tests use a tiny RSVQA-LR subset bundled under `data/benchmarks/`.
- VRSBench grounding subset (12 val / 24 ref + 8 test / 16 ref) materializes on demand:
  `python -m ml.evaluation.prepare_vrsbench_grounding --allow-download` (requires the `grounding` extra + network).
- Phase-2C diagnostics regenerate deterministically: `python -m ml.evaluation.phase2c_diagnostics`.
- Sealed test splits are never used for tuning. `data/` and `models/cache/` are git-ignored — never commit them.

## 9. Troubleshooting / common mistakes

- `python: command not found` → use `python3` or the venv binary; the repo requires 3.11+.
- Wrong interpreter (missing torch/transformers) → `source .venv/bin/activate` and reinstall the needed extra; do not use an unrelated system Python.
- VQA `MODEL_UNAVAILABLE` / capability `UNAVAILABLE_RUNTIME` → checkpoint absent or `MODEL_ROOT` wrong; verify cache path + SHA, keep `ENABLE_REMOTE_NETWORK=false` with a present cache.
- `WORKFLOW_UNSUPPORTED` on a supported query → registry/capability not loaded (check `/api/v1/system/capabilities`); do not add a generic tool fallback.
- `VRSBench ... use --allow-download` → images not materialized; install `[grounding]` and rerun with the flag (needs network).
- Port in use → change `PORT` or stop the other server; prefer one `uvicorn --reload` at a time.
- Node issues → run frontend commands from `apps/web/`; never put API secrets in `VITE_*` variables.

## 10. Team setup

One venv per clone, one `.env` per machine (uncommitted), no shared credentials. There is currently no one-command setup script (future work — not promised). Proposed changes: read `AGENTS.md`, keep diffs minimal and registry-consistent, run focused + full tests and `git diff --check`, no commit/push unless requested.
