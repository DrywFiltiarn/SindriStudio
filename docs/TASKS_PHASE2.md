# TASKS_PHASE2.md — Phase 2: Real Inference
# Goal: Working text-to-image generation with Z-Image-Turbo and SDXL.
# Image results visible in BloomeryUI. Full VRAM tracking.
#
# Prerequisites: ALL Phase 1 tasks complete and CI green.

---

## TASK INDEX

| ID      | Description                                          | Prerequisites   |
|---------|------------------------------------------------------|-----------------|
| P2-A1   | Python worker: thread config + torch setup           | P1 complete     |
| P2-A2   | Python worker: executor DAG engine                   | P2-A1           |
| P2-A3   | Python worker: BaseNode + NODE_REGISTRY              | P2-A2           |
| P2-A4   | Python worker: ZiT pipeline nodes                    | P2-A3           |
| P2-A5   | Python worker: SDXL pipeline nodes                   | P2-A3           |
| P2-A6   | Python worker: wire Execute handler                  | P2-A4           |
| P2-B1   | anvilml-ipc: ImageReady event type                   | P1 complete     |
| P2-B2   | anvilml-core: ArtifactMeta + updated Job type        | P2-B1           |
| P2-B3   | anvilml-server: artifact storage module              | P2-B2           |
| P2-B4   | anvilml-server: GET /v1/artifacts endpoints          | P2-B3           |
| P2-B5   | anvilml-scheduler: handle ImageReady + Completed     | P2-B3           |
| P2-B6   | anvilml-server: job.image_ready WebSocket event      | P2-B5           |
| P2-C1   | BloomeryUI: re-generate types (pnpm generate-types)  | P2-B4           |
| P2-C2   | BloomeryUI: artifacts store + useArtifacts hook      | P2-C1           |
| P2-C3   | BloomeryUI: ArtifactCard + ArtifactGallery           | P2-C2           |
| P2-C4   | BloomeryUI: ZitJobForm component                     | P2-C1           |
| P2-C5   | BloomeryUI: JobProgress component                    | P2-C1           |
| P2-C6   | BloomeryUI: TestPanel update (gallery + progress)    | P2-C3, P2-C4, P2-C5 |
| P2-D1   | Worker install scripts (Linux + Windows)             | P2-A1           |
| P2-D2   | Standalone inference test script                     | P2-A4           |
| P2-E1   | Integration: ZiT end-to-end smoke test               | P2-D2, P2-C6    |
| P2-E2   | Integration: SDXL end-to-end smoke test              | P2-A5, P2-C6    |
| P2-E3   | CI: mock inference in Python worker tests            | P2-A6           |

---

## TASK DETAILS

---

### P2-A1 — Python worker: thread config + torch setup

**Goal:** Worker startup correctly configures all threading parameters
before importing torch. Both real and mock modes verified.

**Files to modify:**
- `backend/worker/worker_main.py`

**Thread configuration order (MUST happen before `import torch`):**
```python
import os

# Set before torch import — these affect OMP and MKL initialization
os.environ.setdefault("OMP_NUM_THREADS",     "14")
os.environ.setdefault("MKL_NUM_THREADS",     "14")
os.environ.setdefault("OPENBLAS_NUM_THREADS","14")
os.environ.setdefault("VECLIB_MAXIMUM_THREADS","14")

# Now safe to import torch
import torch
torch.set_num_threads(14)
torch.set_num_interop_threads(4)
torch.backends.cuda.matmul.allow_tf32 = False   # AMD compatibility
torch.backends.cudnn.allow_tf32 = False
```

Thread counts read from env vars `ANVILML_NUM_THREADS` / `ANVILML_NUM_INTEROP_THREADS`
(defaults: 14 and 4). Rust supervisor sets these based on CPU detection.

**Tests:**
```python
# test: OMP_NUM_THREADS set before torch import (verify env var in worker process)
# test: torch.get_num_threads() returns configured value
# test: allow_tf32 is False after startup
# test: mock mode skips torch import entirely
```

**Acceptance:** pytest exits 0

---

### P2-A2 — Python worker: executor DAG engine

**Goal:** Implement `executor.py` — topological sort + node execution loop.
Does not know about specific node types — purely structural.

**Files to create/modify:**
- `backend/worker/executor.py`
- `backend/worker/tests/test_executor.py`

**Graph format:**
```json
{
  "nodes": [
    {"id": "n1", "type": "ZitLoadPipeline", "inputs": {"model_path": "...", "dtype": "bfloat16"}},
    {"id": "n2", "type": "ZitTextEncode", "inputs": {"pipeline": {"from_node": "n1", "output": "pipeline"}, "prompt": "..."}}
  ]
}
```

**Executor.run_graph(graph, settings, job_id):**
1. Parse nodes and build dependency graph
2. Topological sort (Kahn's algorithm — no external deps)
3. For each node in order:
   - Resolve inputs: literals used directly, `{"from_node": X, "output": Y}` references look up node_outputs[X][Y]
   - Look up type in NODE_REGISTRY, instantiate, call `execute(**inputs)`
   - Cache outputs in `node_outputs[node_id]`
   - Send Progress IPC event
4. Send Completed IPC event

**Tests:**
```python
# test: linear 3-node graph executes in correct order
# test: diamond dependency graph executes correctly
# test: cycle detection raises ValueError
# test: missing node type raises KeyError with helpful message
# test: Progress events sent for each node (mock ipc.write_event)
# test: Completed event sent after all nodes (mock ipc.write_event)
# test: exception in node sends Failed event (mock ipc.write_event)
```

**Acceptance:** pytest exits 0, ≥7 tests pass

---

### P2-A3 — Python worker: BaseNode + NODE_REGISTRY

**Files to create/modify:**
- `backend/worker/nodes/base.py` — BaseNode ABC (complete, from ARCHITECTURE.md)
- `backend/worker/nodes/__init__.py` — NODE_REGISTRY dict
- `backend/worker/tests/test_nodes_base.py`

**Tests:**
```python
# test: BaseNode subclass without NODE_TYPE raises NotImplementedError on registry add
# test: BaseNode subclass with execute() returns dict
# test: NODE_REGISTRY lookup returns correct class
# test: registering duplicate NODE_TYPE raises ValueError
```

**Acceptance:** pytest exits 0

---

### P2-A4 — Python worker: ZiT pipeline nodes

**Goal:** Implement ZitLoadPipeline, ZitTextEncode, ZitSampler, ZitDecode, SaveImage.
See ARCHITECTURE.md for node specifications.

**Files to create:**
- `backend/worker/nodes/zit.py`
- `backend/worker/tests/test_nodes_zit.py`

**Key ZiT implementation notes:**
- `pipe.to("cuda")` — ROCm maps "cuda" via HIP backend. Never use "hip" or "rocm".
- `guidance_scale=0.0` — ZiT-Turbo is distilled, CFG-free. Default and recommended.
- Default steps: 8. Default size: 1024×1024.
- Seed -1 means random: use `torch.randint(0, 2**31, (1,)).item()`
- SaveImage sends `ImageReady` IPC event (not Completed — that's the executor)

**Tests (mock mode — no GPU required):**
```python
# test: ZitLoadPipeline registered in NODE_REGISTRY
# test: ZitSampler with seed=-1 generates a random seed (not -1)
# test: ZitSampler with seed=42 is reproducible (same output tensor shape)
# test: SaveImage calls ipc.write_event with type="ImageReady"
# test: SaveImage image_b64 is valid base64
# test: full 5-node ZiT graph executes without error in mock mode
```

Mock torch behavior: `ANVILML_WORKER_MOCK=1` makes ZitSampler return a
black 1024×1024 PIL Image instead of running inference.

**Acceptance:** `ANVILML_WORKER_MOCK=1 pytest backend/worker/tests/test_nodes_zit.py -v` exits 0

---

### P2-A5 — Python worker: SDXL pipeline nodes

Same pattern as ZiT. Uses `StableDiffusionXLPipeline` from diffusers.
Node types: `SdxlLoadPipeline`, `SdxlTextEncode`, `SdxlSampler`, `SdxlDecode`, `SaveImage` (shared).

Key differences from ZiT:
- SDXL uses two text encoders (prompt + prompt_2)
- guidance_scale default: 7.5 (not distilled)
- Default steps: 20
- Supports negative_prompt

**Tests:** same mock pattern as ZiT.

**Acceptance:** pytest exits 0

---

### P2-A6 — Python worker: wire Execute handler

**Goal:** Connect the `Execute` IPC message to the real executor.

**Files to modify:**
- `backend/worker/worker_main.py`

**Acceptance:**
- [ ] Submitting a ZiT job via the API results in the executor running
- [ ] Progress IPC events received by Rust server
- [ ] ImageReady IPC event received by Rust server
- [ ] Completed IPC event received

---

### P2-B1 — anvilml-ipc: ImageReady event type

**Files to modify:**
- `crates/anvilml-ipc/src/messages.rs` — add `ImageReady` variant to `WorkerEvent`

Fields from IPC_PROTOCOL.md. Add round-trip test.

**Acceptance:** `cargo test -p anvilml-ipc` exits 0

---

### P2-B2 — anvilml-core: ArtifactMeta + updated types

**Files to modify:**
- `crates/anvilml-core/src/types/model.rs` — complete ArtifactMeta struct
- `crates/anvilml-core/src/types/job.rs` — add artifact_count field to Job
- `crates/anvilml-core/src/types/events.rs` — add WsJobImageReady variant

**Acceptance:** `cargo test -p anvilml-core` exits 0

---

### P2-B3 — anvilml-server: artifact storage

**Files to create:**
- `crates/anvilml-server/src/artifact/store.rs`
- `backend/migrations/002_artifacts.sql`

**ArtifactStore:**
- `save(job_id, image_b64, meta) -> Result<ArtifactMeta>`
  Decode base64 → compute SHA256 → write PNG to `{artifact_dir}/{sha256[0..2]}/{sha256}.png`
- `get(hash) -> Result<Option<ArtifactMeta>>`
- `list(job_id: Option<Uuid>) -> Result<Vec<ArtifactMeta>>`

**Tests (use tempdir):**
```rust
// test: save writes file to correct path
// test: save records in DB
// test: get retrieves by hash
// test: list filtered by job_id returns only matching artifacts
// test: save is idempotent (same image twice → same hash, no duplicate)
```

**Acceptance:** `cargo test -p anvilml-server --features mock-hardware` exits 0

---

### P2-B4 — anvilml-server: artifact endpoints

**Files to create:**
- `crates/anvilml-server/src/handlers/artifacts.rs`

Implements `GET /v1/artifacts` and `GET /v1/artifacts/:hash` (file serve).
Re-run `cargo run -p anvilml-openapi` to update openapi.json.

**Tests:**
```rust
// test: GET /v1/artifacts returns empty list initially
// test: GET /v1/artifacts/:hash returns 404 for unknown hash
// test: GET /v1/artifacts/:hash returns PNG bytes with correct Content-Type
```

**Acceptance:** `cargo test --features mock-hardware` exits 0, openapi.json updated

---

### P2-B5 — anvilml-scheduler: handle ImageReady + Completed

**Files to modify:**
- `crates/anvilml-scheduler/src/scheduler.rs`

On `ImageReady` event: call `artifact_store.save()`, emit `WsJobImageReady`.
On `Completed` event: set job status to Completed, update DB.

**Acceptance:** `cargo test --features mock-hardware` exits 0

---

### P2-B6 — anvilml-server: job.image_ready WebSocket event

Ensure `WsJobImageReady` is broadcast to all WS clients when ImageReady received.

**Tests:**
```rust
// test: ImageReady IPC event produces job.image_ready WS event
// test: job.image_ready contains correct image_url and artifact_hash
```

**Acceptance:** `cargo test --features mock-hardware` exits 0

---

### P2-C1 through P2-C6 — BloomeryUI inference UI

Each is a separate task. Run `pnpm generate-types` at start of P2-C1.

**P2-C1:** Regenerate types. Verify `pnpm type-check` passes.
**P2-C2:** `artifacts.store.ts` + `useArtifacts.ts` hook. Tests: store updates on event.
**P2-C3:** `ArtifactCard` + `ArtifactGallery`. Renders image via `<img src>`. Tests: renders empty state + with artifact.
**P2-C4:** `ZitJobForm`. Builds correct 5-node graph JSON. Tests: form validation + graph builder function.
**P2-C5:** `JobProgress`. Shows progress bar during inference. Tests: renders correct percent.
**P2-C6:** TestPanel updated. Gallery and progress visible. `pnpm build` exits 0.

---

### P2-D1 — Worker install scripts

**Files to create:**
- `backend/scripts/install_worker_deps.sh` (Linux)
- `backend/scripts/install_worker_deps.ps1` (Windows)

Both scripts: create venv, install PyTorch ROCm/CUDA/CPU wheel, install requirements.txt.
Detect platform and GPU type automatically.

**Acceptance:** scripts are executable and documented in README.md

---

### P2-D2 — Standalone inference test script

**Files to create:**
- `backend/scripts/test_inference.py`

CLI: `--model-type zit|sdxl --model-path <path> --prompt <text> --output <path>`
Runs inference directly (no IPC, no server). Prints timing and VRAM usage.
Used for debugging inference issues in isolation from the server.

**Acceptance:** script runs without error (requires real model file)

---

### P2-E1 — Integration: ZiT end-to-end

Manual smoke test sequence documented in report:
1. Download z-image-turbo to models/diffusion/
2. Start sindristudio, connect BloomeryUI
3. Submit ZiT job via ZitJobForm
4. Verify image appears in gallery
5. Verify VRAM usage reported correctly
6. Second job faster (model cached)

---

### P2-E2 — Integration: SDXL end-to-end

Same as E1 but with SDXL model. Verify both models can run sequentially.

---

### P2-E3 — CI: mock inference tests

**Goal:** Python worker inference tests run in CI without GPU.

**Files to modify:**
- `backend/worker/tests/test_nodes_zit.py` — all tests work with `ANVILML_WORKER_MOCK=1`
- `.github/workflows/ci.yml` — add `ANVILML_WORKER_MOCK=1` to python-worker job

**Acceptance:** CI green with mock inference tests

---

## PHASE 2 OUT OF SCOPE

- Node graph canvas (Phase 3)
- LoRA loading (Phase 3)
- Multi-GPU routing (Phase 3)
- img2img (Phase 3)
- TunableOp cache warming (Phase 3)
- Authentication (Phase 4)
