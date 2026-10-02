# SATQuery AI — Coding Agent Handoff

Repository:
- https://github.com/bishuk-dev/SIH-26167-SATQuery
- Local: `~/Hackathon Projects/SIH-26167-SATQuery`
- Project: SIH-26167 SatQuery AI

## Goal

Finish the remaining bugs without rewriting the architecture. The immediate goal is to make the canonical `SINGLE_VQA` workflow work end-to-end using the existing frozen SmolVLM implementation, while preserving the registry/planner/execution/security design.

## Environment

Activate:

```bash
cd ~/Hackathon\ Projects
source .venv/bin/activate
cd SIH-26167-SATQuery
```

Use the venv Python 3.12.3. Do NOT use the separate `python314` executable; it does not have PyTorch.

Run API:

```bash
uvicorn apps.api.app.main:app --reload
```

Swagger: `http://127.0.0.1:8000/docs`

## Architecture rules

This is an evidence-grounded remote-sensing system.

Important boundaries:
- Scientific/numeric/spatial claims must come from registered deterministic/model/GIS outputs.
- Natural-language text must not create scientific evidence.
- Planner may select only registered tools/models.
- Execution must go through registered adapters.
- Do not introduce arbitrary Python, shell, SQL, URLs, filesystem paths, model IDs, or tool IDs.
- Do not bypass the planner/registry/execution architecture.
- Make the smallest architecture-consistent change.

## Verified ML state

Pinned SmolVLM:

```text
registry id: smolvlm_256m_instruct_v1
HF model: HuggingFaceTB/SmolVLM-256M-Instruct
revision: 7e3e67edbbed1bf9888184d9df282b700a323964
checkpoint: model.safetensors
SHA256: 74dea5904032e5ae99a2e0eef5179e6ac0f1dedc3ab0c7c2a5d4d387c843203e
preprocessing: smolvlm_single_image_v1
frozen: true
allow_remote_code: false
max_new_tokens: 16
```

Correct checkpoint exists at:

```text
models/cache/models--HuggingFaceTB--SmolVLM-256M-Instruct/snapshots/7e3e67edbbed1bf9888184d9df282b700a323964/model.safetensors
```

Do not change model revisions, SHA verification, or download random alternatives.

## Phase 2A is already validated

Commands:

```bash
python -m ml.evaluation.rsvqa_lr_subset
python -m ml.evaluation.run_phase2a_baseline --allow-download
```

Results:

```text
subset: {"train":14,"validation":4,"test":6}

normalized_exact_match: 0.0
majority_answer: "yes"
majority_normalized_exact_match: 0.6666666666666666
```

These match the documented README baseline exactly. Do not modify this benchmark just to improve the score.

## Test baseline

Initial/full pytest:

```text
733 collected
730 passed
3 failed
3 warnings
```

The three pre-existing failures are:

```text
tests/ml/test_phase2c.py::test_committed_diagnostic_matches_generator
tests/ml/test_phase3a_grounding.py::test_committed_vrsbench_manifest_is_scene_safe_and_reproducible
tests/phase4/test_phase4_closeout.py::test_every_referenced_artifact_exists_and_hash_matches
```

Reasons:
1. Phase 2C committed diagnostic differs from generated diagnostic/source manifest SHA.
2. Phase 3A VRSBench images are missing; test mentions `--allow-download`.
3. Phase 4 `experiments/phase4_temporal_analytics/p4_e01/manifest.json` hash mismatch.

Treat these as separate pre-existing data/artifact consistency problems. Do not delete tests or blindly regenerate artifacts.

## Readiness bug already fixed

`src/satquery/observability.py`/`satquery/observability.py` readiness logic was corrected to find Hugging Face cache using:

```python
registration.model_id
```

rather than the internal registry ID.

Correct cache pattern:

```text
root/cache/models--<HF model id>/snapshots/<revision>/<checkpoint>
```

An accidental `========` line once caused syntax errors; it has been removed.

After the fix:

```text
730 passed, 3 failed, 3 warnings
```

API status became:

```text
registered_count: 6
verified_count: 1
missing_count: 5
invalid_count: 0
```

This fix is correct. Do not undo it.

## Observation already validated

Synthetic GeoTIFF:

```text
/tmp/satquery-api-test.tif
3 bands
128x128
uint8
EPSG:4326
```

Uploaded successfully to:

```text
POST /api/v1/observations
```

Observation:

```text
obs_4e64376ceb414f0fb5730edf37d1112b
```

Status: READY.

Original SHA256:

```text
093faac0d9d38ce5e55830cea97b23b77397c85fd19208b704972dbcbd4e4643
```

Visualization COG SHA256:

```text
fd51be6a0d4f74e842d30a1e72dbb24ef1a13c8ec8da1dfe87ee72b3cd96fe8d
```

## Existing VQA implementation

File:

```text
satquery/inference/vqa.py
```

Contains:
- `VqaBackend`
- `SmolVlmBackend`
- `SingleImageVqaService`

`SingleImageVqaService.answer(observation_id, question)`:
1. validates question
2. loads observation
3. preprocesses visualization
4. invokes backend
5. returns structured `VqaEvidence`

Evidence contains model provenance, prediction, source observations/modalities, domain assessment, warnings and provenance.

The legacy endpoint already exists at:

```text
/api/vqa
```

but the desired solution is canonical `/api/v1/query` integration.

## Interpreter/feasibility already support VQA

`DeterministicQueryInterpreter` recognizes:

```text
"What is visible in this image?"
"How many buildings are there?"
"What is this image?"
```

as:

```text
SINGLE_VQA
```

`satquery/agent/feasibility.py` maps:

```text
SINGLE_VQA -> ("single_image_vqa",)
```

## Current canonical VQA failure

This request was tested:

```json
{
  "query": "What is visible in this image?",
  "observation_ids": ["obs_4e64376ceb414f0fb5730edf37d1112b"]
}
```

Initially:

```text
MODEL_UNAVAILABLE
No available registered capability can execute this request.
```

That was fixed by the readiness correction.

Now capability is:

```text
single_image_vqa
AVAILABLE_WITH_LIMITS
source_phase: 2
tool_ids: []
model_ids: ["smolvlm_256m_instruct_v1"]
executor: null
contract_status: ACCEPTED
runtime_status: PROBED
promotion_status: PROMOTED
```

The same query now fails with:

```text
WORKFLOW_UNSUPPORTED
No bounded registered workflow supports this request.
```

THIS is the main current blocker.

## Planner

File:

```text
satquery/agent/planner.py
```

Current `_POLICIES` only contains policies for:

```text
CHANGE_LOCALIZE + SAR
CHANGE_MEASURE + SAR + area
```

`_policy_for()` raises:

```text
WORKFLOW_UNSUPPORTED
No bounded registered workflow supports this request.
```

Need to add an explicit bounded `SINGLE_VQA` workflow.

Do NOT add a generic arbitrary-tool fallback.

## Tool executor enum

File:

```text
satquery/registry/models.py
```

Existing executors were:

```python
SAR_TEMPORAL_CHANGE = "sar_temporal_change"
MASK_AGREEMENT = "mask_agreement"
MASK_AREA = "mask_area"
```

User has just added:

```python
SINGLE_IMAGE_VQA = "single_image_vqa"
```

Verified:

```bash
python -m py_compile satquery/registry/models.py
python -m pytest tests/registry/test_tools.py -q
```

Result:

```text
13 passed in 0.51s
```

## Tool registry gap

`tools.yaml` currently has:

```text
sar_temporal_change_v1
mask_agreement_v1
compute_mask_area_v1
```

There is no:

```text
single_image_vqa_v1
```

Need to add a bounded VQA tool registration.

Before editing, inspect:

```bash
sed -n '1,260p' satquery/registry/tools.py
```

Do not invent a free-form parameter type if the registry does not support one.

## Execution context / question flow

`ExecutionContext` contains:

```python
metadata: dict[str, Any]
repository: MetadataRepository | None
```

Job runner loads persisted job metadata and constructs:

```python
context = ExecutionContext(
    job_id=...,
    analysis_id=...,
    repository=self.repository,
    cancel_event=...,
    metadata=metadata,
)
```

The analysis payload already contains:

```python
"query": payload.query
```

but the queued job currently has metadata approximately:

```python
"metadata": {
    "verified_common_grid_pairs": verified_pairs
}
```

Therefore, if the VQA adapter needs the original natural-language question, the likely safe change is to add:

```python
"query": payload.query
```

to job metadata.

Do NOT put arbitrary query text into the bounded `PlanStep` unless the registry explicitly supports and validates it.

Do NOT put live service objects into persisted job metadata.

## Existing adapters

File:

```text
satquery/execution/adapters.py
```

Current adapters:
- `SarTemporalChangeAdapter`
- `MaskAreaAdapter`
- `MaskAgreementAdapter`

Signature:

```python
execute(call: ToolCall, context: ExecutionContext) -> ToolResult
```

Existing helpers include:
- `_observation(context, observation_id)`
- `_verify_source_asset(observation)`
- `_require_parameters(call, required)`
- `_verified_common_grid(...)`
- `_resolve_prior_evidence(...)`

Observation lookup already uses `context.repository` and resolves relative asset paths through repository `data_root`.

Inspect before editing:

```bash
sed -n '300,520p' satquery/execution/adapters.py
```

## Adapter constructor constraint

Current constructor type is approximately:

```python
AdapterConstructor = Callable[[ToolRegistration, ArtifactStore | None], ToolAdapter]
```

Default constructors exist for the three current executors.

`build_registered_adapters()` verifies every registered executor has a constructor.

Add VQA without breaking existing tests/custom constructor behavior.

Before changing this, inspect the bottom of `adapters.py`.

## Existing application-level VQA service

`apps/api/app/main.py` already creates:

```text
SingleImageVqaService
```

and stores it in:

```text
application.state.single_image_vqa_service
```

The execution engine is also created there.

Avoid accidentally loading the SmolVLM checkpoint twice or creating an unsafe global.

Inspect the existing dependency construction before deciding how the adapter accesses the service.

## Important design target

Desired flow:

```text
/api/v1/query
  -> DeterministicQueryInterpreter
  -> SINGLE_VQA
  -> FeasibilityValidator
  -> BoundedPlanner
  -> registered single_image_vqa_v1 tool
  -> VQA adapter
  -> existing SingleImageVqaService
  -> structured VqaEvidence
  -> job/analysis completion
```

Do not create a second unrelated VQA pipeline.

## First commands for the coding agent

Before editing anything, run:

```bash
sed -n '1,260p' satquery/registry/tools.py
```

```bash
sed -n '300,520p' satquery/execution/adapters.py
```

```bash
sed -n '1,300p' satquery/agent/planner.py
```

```bash
sed -n '1,280p' satquery/inference/vqa.py
```

```bash
grep -R "SINGLE_VQA\|single_image_vqa\|WORKFLOW_UNSUPPORTED" -n tests satquery apps --exclude-dir=__pycache__
```

Then implement incrementally.

## Testing requirements

After each meaningful change run focused tests.

At minimum:

```bash
python -m pytest tests/registry/test_tools.py -q
```

Then relevant planner/execution/VQA tests.

Then:

```bash
python -m pytest
```

Do not hide the three pre-existing artifact/data failures.

Finally start the API and test the real canonical query:

```json
{
  "query": "What is visible in this image?",
  "observation_ids": ["obs_4e64376ceb414f0fb5730edf37d1112b"]
}
```

The exact model answer is not predetermined. What matters is that execution reaches the VQA service and produces structured evidence rather than `WORKFLOW_UNSUPPORTED`.

## Definition of done

VQA is complete only when:

1. `single_image_vqa_v1` is registered.
2. Runtime capability has a registered VQA tool rather than `tool_ids=[]`.
3. `SINGLE_VQA` gets a bounded plan.
4. Execution uses a registered VQA adapter.
5. Adapter reaches `SingleImageVqaService`.
6. Pinned SmolVLM checkpoint is used.
7. Observation is loaded through controlled repository/store logic.
8. Structured `VqaEvidence` is produced.
9. Canonical query no longer fails at planner selection.
10. Existing tests remain intact.
11. Security boundaries remain intact.
12. Real API end-to-end test has been run.

## Rules for the agent

- Inspect actual files before editing.
- Do not rewrite the project.
- Do not change pinned model/dataset revisions.
- Do not weaken SHA verification.
- Do not disable readiness checks.
- Do not remove failing tests.
- Do not claim success without running the relevant tests.
- Do not claim end-to-end success without actually making the API request.
- Report files changed, why, tests run/results, API result, and remaining failures.
