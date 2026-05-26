# TASKS_PHASE1.md — Phase 1: Foundation
# Goal: Running AnvilML server + connected BloomeryUI test panel.
# Real hardware detection, worker IPC handshake, all REST endpoints,
# WebSocket event stream. No real inference yet.
#
# Task sizing: each task = one testable unit, completable in one Cline session.
# Prerequisites are strict — do not start a task until its prerequisites pass.
#
# Format: each task defines what to implement, what tests to write, and
# what commands must exit 0 before the task can be committed.

---

## TASK INDEX

| ID      | Description                                      | Prerequisites |
|---------|--------------------------------------------------|---------------|
| P1-A1   | Repository scaffold + CI workflow                | none          |
| P1-A2   | Cargo workspace + crate skeletons                | P1-A1         |
| P1-A3   | anvilml-core: config types                       | P1-A2         |
| P1-A4   | anvilml-core: job + model + hardware types       | P1-A3         |
| P1-A5   | anvilml-ipc: message types + framing             | P1-A4         |
| P1-A6   | anvilml-hardware: detector trait + CPU impl      | P1-A4         |
| P1-A7   | anvilml-hardware: ROCm detection                 | P1-A6         |
| P1-A8   | anvilml-hardware: CUDA + IPEX detection          | P1-A6         |
| P1-A9   | anvilml-hardware: mock implementation            | P1-A6         |
| P1-B1   | anvilml-registry: model scanner                  | P1-A4         |
| P1-B2   | anvilml-registry: SQLite persistence             | P1-B1         |
| P1-C1   | Python worker: ipc.py framing                    | P1-A5         |
| P1-C2   | Python worker: worker_main.py startup + Ready    | P1-C1         |
| P1-C3   | Python worker: Ping/Pong + MemoryReport + Shutdown | P1-C2       |
| P1-D1   | anvilml-worker: WorkerPool + spawn               | P1-A9, P1-C2  |
| P1-D2   | anvilml-worker: IPC bridge (stdin writer + stdout reader) | P1-D1  |
| P1-D3   | anvilml-worker: ROCm/CUDA/IPEX env injection     | P1-D1         |
| P1-E1   | anvilml-scheduler: JobQueue + VramLedger         | P1-A4         |
| P1-E2   | anvilml-scheduler: JobScheduler dispatch loop    | P1-E1, P1-D2  |
| P1-F1   | anvilml-server: AppState + router skeleton       | P1-E2         |
| P1-F2   | anvilml-server: GET /health + GET /v1/system     | P1-F1         |
| P1-F3   | anvilml-server: jobs CRUD endpoints              | P1-F1         |
| P1-F4   | anvilml-server: models + workers endpoints       | P1-F1         |
| P1-F5   | anvilml-server: WebSocket /v1/events             | P1-F1         |
| P1-F6   | anvilml-server: system.stats broadcast tick      | P1-F5         |
| P1-G1   | anvilml-openapi: spec generation binary          | P1-F6         |
| P1-H1   | Launcher binary: sindristudio entry point        | P1-F6         |
| P1-H2   | Launcher: browser open + graceful shutdown       | P1-H1         |
| P1-I1   | BloomeryUI: project scaffold + toolchain         | P1-A1         |
| P1-I2   | BloomeryUI: API client + generated types         | P1-G1, P1-I1  |
| P1-I3   | BloomeryUI: Zustand stores                       | P1-I2         |
| P1-I4   | BloomeryUI: connection hooks + polling           | P1-I3         |
| P1-I5   | BloomeryUI: layout components                    | P1-I1         |
| P1-I6   | BloomeryUI: SystemStats + WorkerList components  | P1-I3         |
| P1-I7   | BloomeryUI: JobSubmit + JobList components       | P1-I3         |
| P1-I8   | BloomeryUI: ModelList + EventLog components      | P1-I3         |
| P1-I9   | BloomeryUI: TestPanel page assembly              | P1-I5, P1-I6, P1-I7, P1-I8 |
| P1-J1   | GitHub Actions CI workflow                       | P1-G1, P1-I9  |
| P1-J2   | Integration smoke test                           | P1-H2, P1-I9  |

---

## TASK DETAILS

---

### P1-A1 — Repository scaffold

**Goal:** Create SindriStudio root repo structure, model directories, gitignore.
This is the scaffolding only — no application code.

**Files to create (SindriStudio root):**
- `.gitmodules` — registers backend and frontend submodules
- `.gitignore` — see content below
- `README.md` — project overview
- `models/clip/.gitkeep`
- `models/diffusion/.gitkeep`
- `models/vae/.gitkeep`
- `models/lora/.gitkeep`
- `models/controlnet/.gitkeep`
- `models/unet/.gitkeep`
- `models/upscale/.gitkeep`
- `artifacts/.gitkeep`
- `.cline/state/CURRENT_TASK.md` — initialized to P1-A2
- `.cline/reports/P1-A1.md` — this task's report

**.gitignore content:**
```
# Generated artifacts
artifacts/

# Environment files
.env
*.env

# Editor
.vscode/settings.json
.idea/

# OS
.DS_Store
Thumbs.db
```

**.gitmodules content:**
```
[submodule "backend"]
    path = backend
    url = https://github.com/DrywFiltiarn/AnvilML.git
    branch = develop

[submodule "frontend"]
    path = frontend
    url = https://github.com/DrywFiltiarn/BloomeryUI.git
    branch = develop
```

**Tests:** No tests (scaffold only).
`git status` shows all files tracked. `git submodule status` shows submodules registered.

**Acceptance:**
- [ ] `git submodule status` shows backend and frontend registered
- [ ] `models/` has 7 subdirectories with .gitkeep
- [ ] `artifacts/` directory exists
- [ ] `.cline/state/CURRENT_TASK.md` exists and points to P1-A2

**Commit scope:** root only (no submodule code yet)

---

### P1-A2 — Cargo workspace + crate skeletons

**Goal:** Create AnvilML repo structure: workspace Cargo.toml and all 8 crate
skeletons that compile. Each crate has a `lib.rs` (or `main.rs`) that compiles
with only a `//! crate description` comment. No logic yet.

**Files to create (backend/):**
- `Cargo.toml` — workspace manifest (see ARCHITECTURE.md)
- `Cargo.lock` — generated
- `.gitignore` — `/target`, `anvilml.toml`, `anvilml.db`, `.venv/`, `openapi.json`
- `anvilml.toml.example` — from ENVIRONMENT.md
- `README.md`
- `crates/anvilml-core/Cargo.toml`
- `crates/anvilml-core/src/lib.rs`
- `crates/anvilml-hardware/Cargo.toml`
- `crates/anvilml-hardware/src/lib.rs`
- `crates/anvilml-registry/Cargo.toml`
- `crates/anvilml-registry/src/lib.rs`
- `crates/anvilml-ipc/Cargo.toml`
- `crates/anvilml-ipc/src/lib.rs`
- `crates/anvilml-worker/Cargo.toml`
- `crates/anvilml-worker/src/lib.rs`
- `crates/anvilml-scheduler/Cargo.toml`
- `crates/anvilml-scheduler/src/lib.rs`
- `crates/anvilml-server/Cargo.toml`
- `crates/anvilml-server/src/lib.rs`
- `crates/anvilml-openapi/Cargo.toml`
- `crates/anvilml-openapi/src/main.rs`
- `src/main.rs` — launcher skeleton: `fn main() { println!("SindriStudio"); }`

**Tests:** Each crate has a `#[cfg(test)] mod tests {}` with one trivial test:
```rust
#[test]
fn crate_compiles() { assert!(true); }
```

**Acceptance:**
- [ ] `cargo build --workspace` exits 0
- [ ] `cargo test --workspace` exits 0 (9 tests pass — one per crate + launcher)
- [ ] `cargo clippy --workspace -- -D warnings` exits 0

---

### P1-A3 — anvilml-core: config types

**Goal:** Implement `ServerConfig`, `RocmConfig`, `ModelDirConfig`, `ModelKind`
in `anvilml-core`. Config loads from TOML and environment variables.

**Files to create:**
- `crates/anvilml-core/src/config.rs`
- `crates/anvilml-core/src/error.rs`

**Files to modify:**
- `crates/anvilml-core/src/lib.rs` — `pub mod config; pub mod error;`
- `crates/anvilml-core/Cargo.toml` — add: `config`, `serde`, `thiserror`, `anyhow`

**Config structure:**
```rust
pub struct ServerConfig {
    pub host: String,               // default: "127.0.0.1"
    pub port: u16,                  // default: 8188
    pub database_url: String,       // default: "sqlite://./anvilml.db"
    pub artifact_dir: PathBuf,
    pub python_bin: PathBuf,        // .venv/bin/python or .venv/Scripts/python.exe
    pub worker_script: PathBuf,
    pub max_workers_per_device: usize, // default: 1
    pub vram_budget_fraction: f64,  // default: 0.90
    pub open_browser: bool,         // default: true
    pub model_dirs: Vec<ModelDirConfig>,
    pub hardware: HardwareOverrideConfig,
    pub rocm: RocmConfig,
}

pub struct ModelDirConfig { pub path: PathBuf, pub kind: ModelKind }

pub enum ModelKind { Clip, Diffusion, Vae, Lora, ControlNet, Unet, Upscale }

pub struct RocmConfig {
    pub target_gfx: String,
    pub force_hipblaslt: bool,
    pub flash_attention: bool,
    pub override_gfx_version: Option<String>,
}

pub struct HardwareOverrideConfig {
    pub force_device_type: Option<String>,  // "rocm" | "cuda" | "ipex" | "cpu"
}
```

`impl ServerConfig { pub fn load() -> anyhow::Result<Self> }` — uses `config` crate,
reads `anvilml.toml` then `ANVILML_*` env vars with `_` separator.

**Tests (in config.rs):**
```rust
// test: default config has correct port
// test: env var ANVILML_PORT overrides port
// test: ModelKind serializes/deserializes correctly
// test: load() succeeds with a temp toml file (use tempfile crate)
```

**Acceptance:**
- [ ] `cargo test -p anvilml-core` exits 0, ≥4 tests pass
- [ ] `cargo clippy -p anvilml-core -- -D warnings` exits 0

---

### P1-A4 — anvilml-core: job + model + hardware types

**Goal:** Implement all shared domain types: Job, ModelMeta, GpuDevice, etc.
These are the types that flow through the entire system.

**Files to create:**
- `crates/anvilml-core/src/types/mod.rs`
- `crates/anvilml-core/src/types/job.rs`
- `crates/anvilml-core/src/types/model.rs`
- `crates/anvilml-core/src/types/hardware.rs`
- `crates/anvilml-core/src/types/worker.rs`
- `crates/anvilml-core/src/types/events.rs`

**Files to modify:**
- `crates/anvilml-core/src/lib.rs` — add `pub mod types;`
- `crates/anvilml-core/Cargo.toml` — add: `uuid`, `chrono`, `serde_json`, `utoipa`

**Key types to implement** (all derive `Serialize, Deserialize, Clone, Debug, ToSchema`):

job.rs: `Job`, `JobStatus` (enum), `JobSettings`, `SubmitJobRequest`, `SubmitJobResponse`
model.rs: `ModelMeta`, `ModelKind` (re-export from config), `ArtifactMeta`, `DType` enum
hardware.rs: `GpuDevice`, `DeviceType` (enum: Rocm/Cuda/Ipex/Cpu), `HardwareInfo`,
             `HostInfo`, `InferenceCaps`
worker.rs: `WorkerInfo`, `WorkerStatus` (enum), `LoadedModel`
events.rs: `WsEvent` enum (all variants from API_CONTRACT.md), `WsEventBase`

**Tests:**
```rust
// test: JobStatus serializes to expected string values
// test: DeviceType serializes to expected string values
// test: Job round-trips through serde_json
// test: WsEvent discriminated union deserializes by "type" field
```

**Acceptance:**
- [ ] `cargo test -p anvilml-core` exits 0, ≥8 tests pass
- [ ] `cargo clippy -p anvilml-core -- -D warnings` exits 0

---

### P1-A5 — anvilml-ipc: message types + framing

**Goal:** Implement `WorkerMessage`, `WorkerEvent`, and async framing functions.
This is the binary contract — field names must exactly match IPC_PROTOCOL.md.

**Files to create:**
- `crates/anvilml-ipc/src/messages.rs`
- `crates/anvilml-ipc/src/framing.rs`

**Files to modify:**
- `crates/anvilml-ipc/src/lib.rs` — `pub mod messages; pub mod framing;`
- `crates/anvilml-ipc/Cargo.toml` — add: `anvilml-core`, `rmp-serde`, `tokio`, `serde`

**Implement** all `WorkerMessage` and `WorkerEvent` variants from IPC_PROTOCOL.md.
`framing.rs`: `write_message<W, T>` and `read_message<R, T>` (async, generic).

**Tests:**
```rust
// test: WorkerMessage::Ping { seq: 42 } serializes and deserializes correctly
// test: WorkerEvent::Ready { ... } round-trips through rmp_serde
// test: WorkerMessage::Execute fields match IPC_PROTOCOL.md exactly (field name assertions)
// test: framing round-trip — write then read recovers original message
// test: message larger than 64MiB is rejected by read_message
```

**Acceptance:**
- [ ] `cargo test -p anvilml-ipc` exits 0, ≥5 tests pass
- [ ] All WorkerMessage and WorkerEvent variants implemented (count matches IPC_PROTOCOL.md)

---

### P1-A6 — anvilml-hardware: detector trait + CPU impl

**Goal:** Define the `HardwareDetector` trait and implement the CPU fallback.
All detection results produce `HardwareInfo` from `anvilml-core`.

**Files to create:**
- `crates/anvilml-hardware/src/detector.rs` — `HardwareDetector` trait
- `crates/anvilml-hardware/src/cpu.rs` — CPU fallback implementation
- `crates/anvilml-hardware/src/error.rs` — `HardwareError`

**Files to modify:**
- `crates/anvilml-hardware/src/lib.rs`
- `crates/anvilml-hardware/Cargo.toml` — add: `anvilml-core`, `sysinfo`, `anyhow`, `thiserror`, `async-trait`

**Trait:**
```rust
#[async_trait]
pub trait DeviceDetector: Send + Sync {
    async fn detect(&self) -> Result<Vec<GpuDevice>, HardwareError>;
    fn device_type(&self) -> DeviceType;
}
```

`CpuDetector::detect()` — always returns one `GpuDevice` with `DeviceType::Cpu`,
VRAM set to total system RAM, uses `sysinfo` for RAM/CPU info.

**Public API:**
```rust
pub async fn detect_all_devices(config: &HardwareOverrideConfig) -> Result<HardwareInfo>
```
Runs all available detectors, merges results, applies config overrides.

**Tests:**
```rust
// test: CpuDetector always returns exactly one device
// test: CpuDetector device has DeviceType::Cpu
// test: detect_all_devices with force_device_type="cpu" returns only CPU device
// test: HardwareInfo.primary_device() returns first non-CPU device if present, else CPU
```

**Acceptance:**
- [ ] `cargo test -p anvilml-hardware` exits 0, ≥4 tests pass

---

### P1-A7 — anvilml-hardware: ROCm detection

**Goal:** Implement ROCm device detection via rocm-smi and rocminfo.
Gracefully handles absence of ROCm tools (returns empty, doesn't panic).

**Files to create:**
- `crates/anvilml-hardware/src/rocm.rs`

**Files to modify:**
- `crates/anvilml-hardware/src/lib.rs` — add `pub mod rocm;`
- `crates/anvilml-hardware/src/detector.rs` — register RocmDetector

**Implement:**
- `RocmDetector::detect()` — calls `rocm-smi --showmeminfo vram --json`, parses output
- `detect_gfx_arch(device_idx)` — calls `rocminfo`, parses GFX string
- `detect_rocm_version()` — reads `/opt/rocm/.info/version` or `rocm-smi --version`
- `detect_rebar()` — parses `lspci -v` for prefetchable regions > 256MB
- `RocmCapabilities` — hipblaslt, aotriton_flash_attn, fp8, bf16, rebar_enabled

Graceful degradation: if `rocm-smi` not found → return `Ok(vec![])` with debug log.

**Tests:**
```rust
// test: RocmDetector returns Ok(vec![]) when rocm-smi is not in PATH
// test: parse_rocm_smi_json correctly extracts VRAM from known JSON fixture
// test: detect_gfx_arch returns None gracefully when rocminfo unavailable
// test: rocm_version_gte("7.0.0", 7, 0) == true
// test: rocm_version_gte("6.3.1", 7, 0) == false
```

Note: tests use fixture strings — they do not require real ROCm installed.

**Acceptance:**
- [ ] `cargo test -p anvilml-hardware` exits 0
- [ ] RocmDetector implements DeviceDetector trait

---

### P1-A8 — anvilml-hardware: CUDA + IPEX detection

**Goal:** Implement CUDA (nvidia-smi) and IPEX (intel-smi/xpu-smi) detection.
Same graceful degradation pattern as ROCm.

**Files to create:**
- `crates/anvilml-hardware/src/cuda.rs`
- `crates/anvilml-hardware/src/ipex.rs`

**Tests for each:** same pattern as P1-A7 — fixture-based, no real GPU needed.

**Acceptance:**
- [ ] `cargo test -p anvilml-hardware` exits 0
- [ ] Both implement DeviceDetector trait

---

### P1-A9 — anvilml-hardware: mock implementation

**Goal:** Implement `MockHardwareDetector` behind the `mock-hardware` feature flag.
Used in CI and in Rust integration tests.

**Files to create:**
- `crates/anvilml-hardware/src/mock.rs`

**Feature gate:**
```rust
// In lib.rs
#[cfg(feature = "mock-hardware")]
pub mod mock;
```

**MockHardwareDetector:**
- Reads `ANVILML_MOCK_DEVICE_TYPE` (default: "cpu")
- Reads `ANVILML_MOCK_VRAM_MIB` (default: 8192)
- Reads `ANVILML_MOCK_GFX_ARCH` (default: "gfx1201")
- Returns a single fake `GpuDevice` with configured values

**Tests (behind feature flag):**
```rust
#[cfg(feature = "mock-hardware")]
mod mock_tests {
    // test: mock returns configured device type
    // test: mock returns configured VRAM
    // test: mock with ANVILML_MOCK_DEVICE_TYPE=rocm returns RocmGpu
}
```

**Acceptance:**
- [ ] `cargo test -p anvilml-hardware --features mock-hardware` exits 0
- [ ] `cargo test -p anvilml-hardware` (no feature) exits 0 (mock code not compiled)

---

### P1-B1 — anvilml-registry: model scanner

**Goal:** Scan configured model directories, produce `Vec<ModelMeta>`.
Walk filesystem, detect file types, infer dtype from filename/size.
No database yet — pure in-memory result.

**Files to create:**
- `crates/anvilml-registry/src/scanner.rs`
- `crates/anvilml-registry/src/registry.rs`

**Files to modify:**
- `crates/anvilml-registry/src/lib.rs`
- `crates/anvilml-registry/Cargo.toml` — add: `anvilml-core`, `walkdir`, `sha2`, `hex`, `anyhow`

**Scanner:**
- Walk dirs recursively, include: `.safetensors`, `.gguf`, `.pt`, `.bin`
- `ModelMeta.id` = first 12 chars of `hex(sha256(relative_path_string))`
- `dtype_hint`: infer from filename keywords (fp16, bf16, f32, int8) or file size heuristic
- `sha256` field: always `None` at scan time (lazy computation)

**VRAM estimate heuristic:**
```rust
pub fn estimate_vram_mib(size_bytes: u64, dtype: DType) -> u64 {
    let multiplier = match dtype {
        DType::Float32 => 1.0,
        DType::Float16 | DType::Bfloat16 => 0.5,
        DType::Unknown => 0.5, // assume half precision
    };
    ((size_bytes as f64 * multiplier) / (1024.0 * 1024.0)) as u64
}
```

**Tests (use tempdir with fake model files):**
```rust
// test: scanner finds .safetensors files recursively
// test: scanner ignores non-model files (.txt, .py, etc.)
// test: estimate_vram_mib(1_000_000_000, Float16) == 476 (approx)
// test: dtype_hint inferred from filename "flux1-dev-fp16.safetensors" == Float16
// test: empty directory returns empty Vec
```

**Acceptance:**
- [ ] `cargo test -p anvilml-registry` exits 0, ≥5 tests pass

---

### P1-B2 — anvilml-registry: SQLite persistence

**Goal:** Persist scanned models to SQLite. Implement `ModelRegistry` with
full CRUD backed by the DB. Integrates scanner from P1-B1.

**Files to create:**
- `crates/anvilml-registry/src/db.rs`
- `crates/anvilml-registry/migrations/001_models.sql`

**Files to modify:**
- `crates/anvilml-registry/src/registry.rs` — full implementation
- `crates/anvilml-registry/Cargo.toml` — add: `sqlx`

**ModelRegistry public API:**
```rust
pub struct ModelRegistry { /* db pool + in-memory cache */ }

impl ModelRegistry {
    pub async fn new(pool: SqlitePool) -> Result<Self>
    pub async fn scan_and_persist(&self, dirs: &[ModelDirConfig]) -> Result<()>
    pub async fn list(&self, kind: Option<ModelKind>) -> Result<Vec<ModelMeta>>
    pub async fn get(&self, id: &str) -> Result<Option<ModelMeta>>
    pub async fn count(&self) -> Result<usize>
    pub fn estimate_vram_mib(&self, path: &str, dtype: DType) -> u64
}
```

**Tests (use sqlx in-memory SQLite: `sqlite::memory:`):**
```rust
// test: scan_and_persist with temp dir containing one safetensors file
// test: list() returns all persisted models
// test: get() returns correct model by id
// test: get() returns None for unknown id
// test: scan_and_persist is idempotent (run twice, no duplicate rows)
```

**Acceptance:**
- [ ] `cargo test -p anvilml-registry` exits 0, ≥5 tests pass

---

### P1-C1 — Python worker: ipc.py framing

**Goal:** Implement `ipc.py` — the IPC framing layer for the Python worker.
Exact framing match with `anvilml-ipc` Rust framing (same 4-byte prefix + msgpack).

**Files to create:**
- `backend/worker/ipc.py`
- `backend/worker/tests/__init__.py`
- `backend/worker/tests/conftest.py`
- `backend/worker/tests/test_ipc.py`

**ipc.py implementation:** from IPC_PROTOCOL.md — `read_message()` and `write_event()`.

**Tests (test_ipc.py):**
```python
# test: write_event then read_message round-trips correctly using BytesIO
# test: read_message raises EOFError on empty stream
# test: large message (>1MB) round-trips correctly
# test: write_event flushes stdout (mock stdout, verify flush called)
# test: all WorkerMessage types can be parsed by read_message
```

**Acceptance:**
- [ ] `ANVILML_WORKER_MOCK=1 python -m pytest backend/worker/tests/test_ipc.py -v` exits 0, ≥5 pass

---

### P1-C2 — Python worker: worker_main.py startup + Ready

**Goal:** Implement the main worker entry point. Handles startup sequence:
configure threading → import torch (or mock) → detect device → send Ready event.

**Files to create:**
- `backend/worker/worker_main.py`
- `backend/worker/tests/test_worker_startup.py`

**worker_main.py:**
- CLI args: `--worker-id <uuid>` `--device-index <int>`
- Mock mode: `ANVILML_WORKER_MOCK=1` — skip torch, send fake Ready immediately
- Real mode: set env vars first, then import torch, detect device, send Ready
- Thread config (real mode): set OMP/MKL env vars BEFORE import torch
- After Ready: enter IPC read loop (blocking)

**Tests (test_worker_startup.py):**
```python
# test: mock mode sends Ready event with mock_mode=true
# test: Ready event contains correct worker_id (from CLI arg)
# test: Ready event contains correct device_index (from CLI arg)
# test: Ready event num_threads and num_interop_threads are positive integers
# test: worker exits cleanly when stdin is closed
```
All tests use `ANVILML_WORKER_MOCK=1`.

**Acceptance:**
- [ ] `ANVILML_WORKER_MOCK=1 python -m pytest backend/worker/tests/test_worker_startup.py -v` exits 0

---

### P1-C3 — Python worker: Ping/Pong + MemoryReport + Shutdown

**Goal:** Implement remaining IPC message handlers in the worker loop.

**Files to modify:**
- `backend/worker/worker_main.py` — add handlers for Ping, Shutdown, and MemoryReport timer

**Files to create:**
- `backend/worker/tests/test_worker_messages.py`

**Implement:**
- `Ping` → send `Pong { seq: ping.seq }`
- `Shutdown` → send `Dying { reason: "Received Shutdown" }`, exit 0
- MemoryReport timer: background thread sends MemoryReport every 10s

**Tests:**
```python
# test: Ping message produces Pong with same seq
# test: Shutdown message produces Dying event then process exits 0
# test: MemoryReport has correct worker_id
# test: MemoryReport vram fields are non-negative integers
```

**Acceptance:**
- [ ] `ANVILML_WORKER_MOCK=1 python -m pytest backend/worker/tests/test_worker_messages.py -v` exits 0

---

### P1-D1 — anvilml-worker: WorkerPool + spawn

**Goal:** Implement `WorkerPool` — spawns Python worker processes, tracks
per-worker state, provides acquire/release interface.

**Files to create:**
- `crates/anvilml-worker/src/pool.rs`
- `crates/anvilml-worker/src/managed.rs`

**Files to modify:**
- `crates/anvilml-worker/src/lib.rs`
- `crates/anvilml-worker/Cargo.toml` — add: `anvilml-core`, `anvilml-ipc`,
  `anvilml-hardware`, `tokio`, `uuid`, `tracing`, `anyhow`

**WorkerPool:**
```rust
pub struct WorkerPool { /* ... */ }
impl WorkerPool {
    pub async fn new(config: Arc<ServerConfig>, hardware: Arc<HardwareInfo>) -> Result<Self>
    pub async fn spawn_for_device(&self, device: &GpuDevice) -> Result<Uuid>
    pub async fn spawn_initial_workers(&self) -> Result<()>
    pub async fn acquire_idle(&self) -> Option<Uuid>
    pub async fn set_busy(&self, id: Uuid, job_id: Uuid)
    pub async fn set_idle(&self, id: Uuid)
    pub async fn send_to(&self, id: Uuid, msg: WorkerMessage) -> Result<()>
    pub fn subscribe_events(&self) -> broadcast::Receiver<WorkerEvent>
    pub async fn status_snapshot(&self) -> Vec<WorkerInfo>
    pub async fn shutdown_all(&self)
}
```

**Tests (use mock hardware + mock Python worker via `ANVILML_WORKER_MOCK=1`):**
```rust
// test: spawn_for_device creates a ManagedWorker entry
// test: newly spawned worker transitions Initializing → Idle after Ready event
// test: acquire_idle returns None when all workers busy
// test: set_busy/set_idle toggle status correctly
// test: send_to delivers message to correct worker
```

Note: tests spawn a real Python process with `ANVILML_WORKER_MOCK=1`.
Python must be available in PATH for these tests.
If Python unavailable: skip with `#[ignore]` and note in report.

**Acceptance:**
- [ ] `cargo test -p anvilml-worker --features mock-hardware` exits 0

---

### P1-D2 — anvilml-worker: IPC bridge

**Goal:** Implement the async stdin writer task and stdout reader task
for each managed worker. Wire WorkerEvent broadcast channel.

**Files to create:**
- `crates/anvilml-worker/src/ipc_bridge.rs`

**Files to modify:**
- `crates/anvilml-worker/src/pool.rs` — start bridge tasks on spawn
- `crates/anvilml-worker/src/managed.rs` — add tx channel, child handle

**IPC bridge:**
- stdin writer task: receives `WorkerMessage` from mpsc channel, frames + writes to stdin
- stdout reader task: reads from stdout, deserializes `WorkerEvent`, broadcasts
- Both tasks: on pipe close → mark worker Dead, attempt respawn after 2s delay

**Tests:**
```rust
// test: Ping sent to worker produces Pong received on broadcast channel
// test: worker marked Dead when Python process exits
// test: message to Dead worker returns Err (not panic)
```

**Acceptance:**
- [ ] `cargo test -p anvilml-worker --features mock-hardware` exits 0
- [ ] Ping/Pong round-trip test passes with real mock Python worker

---

### P1-D3 — anvilml-worker: environment injection

**Goal:** Implement per-device environment variable injection.
Each device type (ROCm, CUDA, IPEX) gets the correct env vars injected
into the Python worker process before spawn.

**Files to create:**
- `crates/anvilml-worker/src/env.rs`

**Implement:**
```rust
pub fn build_worker_env(
    device: &GpuDevice,
    config: &ServerConfig,
) -> HashMap<String, String>
```
Returns the full environment map for the worker process.
See ENVIRONMENT.md for the exact variables per device type.

Platform-aware: on Windows, Python binary path uses backslashes,
`HIP_PATH` points to the ROCm wheel location, not `/opt/rocm`.

**Tests:**
```rust
// test: ROCm env includes HIP_VISIBLE_DEVICES="{index}"
// test: ROCm env includes ROCBLAS_USE_HIPBLASLT=1 when force_hipblaslt=true
// test: CUDA env includes CUDA_VISIBLE_DEVICES="{index}"
// test: CPU env does NOT include HIP_VISIBLE_DEVICES
// test: env does not include ANVILML_WORKER_MOCK when not set
```

**Acceptance:**
- [ ] `cargo test -p anvilml-worker --features mock-hardware` exits 0

---

### P1-E1 — anvilml-scheduler: JobQueue + VramLedger

**Goal:** Implement job queue and VRAM tracking data structures.
Pure logic — no async, no I/O. Easy to unit test.

**Files to create:**
- `crates/anvilml-scheduler/src/queue.rs`
- `crates/anvilml-scheduler/src/ledger.rs`

**Files to modify:**
- `crates/anvilml-scheduler/src/lib.rs`
- `crates/anvilml-scheduler/Cargo.toml`

**JobQueue:** priority-ordered `VecDeque<Job>`. Operations: push, pop_front,
cancel(id), get(id), list(), len().

**VramLedger:** tracks per-device loaded models and VRAM usage.
```rust
pub struct VramLedger {
    budget: HashMap<usize, u64>,       // device_index → budget MiB
    loaded: HashMap<usize, HashMap<String, u64>>, // device → model → MiB
}
impl VramLedger {
    pub fn used_mib(&self, device: usize) -> u64
    pub fn free_mib(&self, device: usize) -> u64
    pub fn would_fit(&self, device: usize, additional_mib: u64) -> bool
    pub fn mark_loaded(&mut self, device: usize, model: String, mib: u64)
    pub fn mark_evicted(&mut self, device: usize, model: &str)
}
```

**Tests (12+ tests covering all operations):**
```rust
// test: push then pop_front returns same job
// test: cancel removes job from queue
// test: cancel returns Err for unknown id
// test: ledger used_mib sums correctly
// test: ledger free_mib = budget - used
// test: would_fit returns false when over budget
// test: mark_loaded then mark_evicted → used_mib returns to 0
// test: multiple devices tracked independently
```

**Acceptance:**
- [ ] `cargo test -p anvilml-scheduler` exits 0, ≥8 tests pass

---

### P1-E2 — anvilml-scheduler: JobScheduler dispatch loop

**Goal:** Implement `JobScheduler` — wires queue, ledger, and worker pool.
Dispatch loop runs in background, assigns queued jobs to idle workers.

**Files to create:**
- `crates/anvilml-scheduler/src/scheduler.rs`
- `crates/anvilml-scheduler/src/dag.rs`

**Files to modify:**
- `crates/anvilml-scheduler/src/lib.rs`
- `crates/anvilml-scheduler/Cargo.toml` — add: `anvilml-worker`, `tokio`, `tracing`

**dag.rs:** topological sort for workflow graph JSON.
Input: `{"nodes":[{"id":"n1",...},...]}`
Output: `Vec<String>` of node IDs in execution order, or `Err` if cycle detected.

**JobScheduler:**
```rust
pub struct JobScheduler { /* ... */ }
impl JobScheduler {
    pub fn new(workers: Arc<WorkerPool>, registry: Arc<ModelRegistry>,
               hardware: Arc<HardwareInfo>, db: SqlitePool) -> Self
    pub async fn submit(&self, workflow: Value, settings: JobSettings) -> Result<SubmitJobResponse>
    pub async fn cancel(&self, job_id: Uuid) -> Result<()>
    pub async fn get_job(&self, id: Uuid) -> Option<Job>
    pub async fn list_jobs(&self, status_filter: Option<JobStatus>) -> Vec<Job>
    pub fn start_dispatch_loop(self: &Arc<Self>)
    pub fn stop(&self)
    pub fn subscribe_status(&self) -> broadcast::Receiver<JobStatusEvent>
}
```

**Tests:**
```rust
// test: submit returns job with status Queued
// test: submitted job appears in list_jobs
// test: cancel changes status to Cancelled
// test: dag topological sort on linear graph: n1→n2→n3 returns [n1,n2,n3]
// test: dag cycle detection returns Err
// test: dispatch loop assigns Queued job to Idle worker (mock worker pool)
```

**Acceptance:**
- [ ] `cargo test -p anvilml-scheduler --features mock-hardware` exits 0, ≥6 tests pass

---

### P1-F1 — anvilml-server: AppState + router skeleton

**Goal:** Implement `AppState`, the axum router, and middleware stack.
All route handlers return 501 stubs. Server starts and responds.

**Files to create:**
- `crates/anvilml-server/src/state.rs`
- `crates/anvilml-server/src/router.rs`
- `crates/anvilml-server/src/error.rs`
- `crates/anvilml-server/src/handlers/mod.rs`

**Files to modify:**
- `crates/anvilml-server/src/lib.rs` — implement `pub async fn start(config: ServerConfig) -> Result<()>`
- `crates/anvilml-server/Cargo.toml` — add all deps

**AppState** — `Arc`-wrapped fields: config, scheduler, workers, registry, hardware, db, broadcaster.

**Middleware stack:** CORS (any origin), compression (gzip), tracing, request-id header.

**Tests:**
```rust
// test: server starts on configured port (bind, then immediately stop)
// test: unknown route returns 404 JSON (not HTML)
// test: CORS headers present on response
```

**Acceptance:**
- [ ] `cargo test -p anvilml-server --features mock-hardware` exits 0
- [ ] Server binary starts without error and binds to port

---

### P1-F2 — anvilml-server: GET /health + GET /v1/system

**Goal:** Implement the two read-only diagnostic endpoints.
Shapes must exactly match API_CONTRACT.md.

**Files to create:**
- `crates/anvilml-server/src/handlers/health.rs`
- `crates/anvilml-server/src/handlers/system.rs`

**Tests (integration tests in backend/tests/):**
```rust
// test: GET /health returns 200 with status="ok"
// test: GET /health response includes version field
// test: GET /v1/system returns 200 with devices array
// test: GET /v1/system devices[0].device_type is a valid enum value
// test: GET /v1/system workers array present (may be empty in test)
// test: GET /v1/system queue_depth is a non-negative integer
```

**Acceptance:**
- [ ] `cargo test --features mock-hardware` exits 0
- [ ] `curl http://127.0.0.1:8188/health` returns `{"status":"ok",...}` (manual verification)

---

### P1-F3 — anvilml-server: jobs CRUD endpoints

**Files to create:**
- `crates/anvilml-server/src/handlers/jobs.rs`

Implements: `POST /v1/jobs`, `GET /v1/jobs`, `GET /v1/jobs/:id`, `DELETE /v1/jobs/:id`

**Tests:**
```rust
// test: POST /v1/jobs returns 202 with job_id UUID
// test: POST /v1/jobs with invalid dtype returns 400
// test: GET /v1/jobs returns array
// test: GET /v1/jobs/:id returns 404 for unknown id
// test: GET /v1/jobs/:id returns job after POST
// test: DELETE /v1/jobs/:id returns 200 for queued job
// test: DELETE /v1/jobs/:id returns 404 for unknown id
```

**Acceptance:** `cargo test --features mock-hardware` exits 0

---

### P1-F4 — anvilml-server: models + workers endpoints

**Files to create:**
- `crates/anvilml-server/src/handlers/models.rs`
- `crates/anvilml-server/src/handlers/workers.rs`

Implements: `GET /v1/models`, `GET /v1/models/:id`, `GET /v1/workers`

**Tests:**
```rust
// test: GET /v1/models returns { models: [], total: 0 } when no models
// test: GET /v1/models/:id returns 404 for unknown id
// test: GET /v1/models ?kind=diffusion filters correctly
// test: GET /v1/workers returns workers array
```

**Acceptance:** `cargo test --features mock-hardware` exits 0

---

### P1-F5 — anvilml-server: WebSocket /v1/events

**Files to create:**
- `crates/anvilml-server/src/handlers/events.rs`
- `crates/anvilml-server/src/ws/broadcaster.rs`

**EventBroadcaster:** wraps `broadcast::Sender<WsEvent>`. All subsystems
call `broadcaster.send(event)` to fan out to all connected WS clients.

**events.rs:** WebSocket upgrade handler. On connect: subscribe to broadcaster.
For each event: serialize to JSON text frame. Ping/pong keepalive every 30s.
On client disconnect: drop subscription cleanly.

**Tests:**
```rust
// test: WS connection succeeds (using tokio_tungstenite test client)
// test: event sent to broadcaster arrives at connected WS client
// test: client disconnect does not crash server
// test: multiple clients all receive same broadcast event
```

**Acceptance:** `cargo test --features mock-hardware` exits 0

---

### P1-F6 — anvilml-server: system.stats broadcast tick

**Goal:** Background task that sends `system.stats` WsEvent every 5 seconds.

**Files to modify:**
- `crates/anvilml-server/src/lib.rs` — start stats ticker on server startup

**Tests:**
```rust
// test: system.stats event received within 6 seconds of server start
// test: system.stats has cpu_usage_percent field as f64
// test: system.stats gpu_stats array length matches device count
```

**Acceptance:** `cargo test --features mock-hardware` exits 0

---

### P1-G1 — anvilml-openapi: spec generation

**Goal:** Generate `openapi.json` from annotated server types and handlers.
The generated file is the contract BloomeryUI reads.

**Files to modify:**
- `crates/anvilml-openapi/src/main.rs` — implement generation
- All `anvilml-core` types — add `#[derive(ToSchema)]`
- All handler functions — add `#[utoipa::path(...)]` annotations
- `crates/anvilml-openapi/Cargo.toml` — add utoipa deps

**Tests:**
```rust
// test: generation binary exits 0
// test: openapi.json is valid JSON
// test: openapi.json contains all Phase 1 endpoint paths
// test: openapi.json schemas include "Job", "SystemInfo", "WsEvent"
```

Run generation as part of test: `cargo run -p anvilml-openapi && cargo test -p anvilml-openapi`

**Acceptance:**
- [ ] `cargo run -p anvilml-openapi` exits 0 and produces `backend/openapi.json`
- [ ] `backend/openapi.json` committed to repo

---

### P1-H1 — Launcher binary: sindristudio entry point

**Goal:** Implement `backend/src/main.rs` — the `sindristudio` binary.
Calls into `anvilml-server::start()`.

**Files to modify:**
- `backend/src/main.rs` — full implementation
- `backend/Cargo.toml` (root, not workspace) — add launcher binary deps

**Implementation:**
- Parse CLI args: `--config <path>`, `--port <n>`, `--host <addr>`, `--no-browser`
- Load `ServerConfig` (with CLI overrides taking precedence)
- Initialize tracing subscriber
- Call `anvilml_server::start(config).await`

**Tests:** `cargo test` (build test — integration handled in P1-J2)

**Acceptance:**
- [ ] `cargo build --release` exits 0
- [ ] `./target/release/sindristudio --help` shows usage

---

### P1-H2 — Launcher: browser open + graceful shutdown

**Goal:** Open browser on startup, handle Ctrl+C gracefully.

**Files to modify:**
- `backend/src/main.rs`
- `backend/Cargo.toml` — add `open` crate

**Graceful shutdown:** on SIGINT/SIGTERM → call `workers.shutdown_all()`,
wait up to 10s, then exit.

Browser open: after server binds successfully, call `open::that(url)`.
Skipped if `--no-browser` flag set.

**Tests:**
```rust
// test: --no-browser flag disables browser open (mock open::that)
// test: server responds to shutdown signal within 10s
```

**Acceptance:**
- [ ] `cargo build --release` exits 0
- [ ] Server starts, browser opens (manual verification)

---

### P1-I1 — BloomeryUI: project scaffold

**Goal:** Create BloomeryUI repo structure with Vite, React, TypeScript,
Tailwind, Vitest. Project builds and tests run.

**Files to create:** all from ARCHITECTURE.md frontend section.
Stub components: `<div>{ComponentName}</div>`.
All barrel `index.ts` files with re-exports.

**Tests (initial):**
```typescript
// src/tests/components/StatusBadge.test.tsx — renders without crash
// src/tests/stores/connection.store.test.ts — initial state is correct
```

**Acceptance:**
- [ ] `pnpm install` exits 0
- [ ] `pnpm build` exits 0
- [ ] `pnpm test:run` exits 0
- [ ] `pnpm type-check` exits 0
- [ ] `pnpm lint` exits 0

---

### P1-I2 — BloomeryUI: API client + generated types

**Goal:** Implement `client.ts`, run `pnpm generate-types` to produce
`src/generated/api-types.ts`, implement all endpoint files.

Prerequisite: P1-G1 must be complete (openapi.json must exist).

**Files to implement:**
- `src/api/client.ts` — `apiFetch<T>` wrapper
- `src/api/websocket.ts` — `WebSocketManager` class
- `src/api/endpoints/*.ts` — all 5 endpoint files
- `src/generated/api-types.ts` — generated (committed)

**Tests:**
```typescript
// test: apiFetch throws ApiError on non-2xx response
// test: apiFetch parses JSON response body
// test: WebSocketManager.connect updates connection store
// test: WebSocketManager reconnects on close
```

**Acceptance:**
- [ ] `pnpm generate-types` exits 0
- [ ] `pnpm type-check` exits 0 (generated types used correctly)
- [ ] `pnpm test:run` exits 0

---

### P1-I3 — BloomeryUI: Zustand stores

**Files to implement:** all 4 stores from ARCHITECTURE.md.
All store state transitions tested.

**Tests (≥3 per store):**
```typescript
// connection.store: setStatus updates status, persists URL to localStorage
// system.store: setInfo updates info, null initially
// jobs.store: upsertJob adds and updates correctly, removeJob removes
// events.store: addEvent appends, ring buffer caps at 200
```

**Acceptance:**
- [ ] `pnpm test:run` exits 0, ≥12 store tests pass

---

### P1-I4 — BloomeryUI: connection hooks + polling

**Files to implement:** `useConnection.ts`, `useSystemStats.ts`, `useJobEvents.ts`

**Tests:**
```typescript
// useSystemStats: polling starts on connect, stops on disconnect
// useSystemStats: updates system store with response data
// useJobEvents: WS job.completed event calls jobs.store.upsertJob
```

**Acceptance:** `pnpm test:run` exits 0

---

### P1-I5 through P1-I9 — BloomeryUI Components + TestPanel

These follow the same pattern — implement component, write unit test, verify build.
Each is a separate task (one component group per task).

**P1-I5:** AppShell, Sidebar, TopBar — layout structure, renders without crash
**P1-I6:** SystemStats, VramBar, WorkerList — renders with mock store data
**P1-I7:** JobSubmitForm, JobList, JobDetail — submit calls API, list renders jobs
**P1-I8:** ModelList, EventLog, EventEntry — renders empty state + with data
**P1-I9:** TestPanel page assembly — all sections visible, no console errors

For each: `pnpm test:run` exits 0 before commit.

---

### P1-J1 — GitHub Actions CI workflow

**Goal:** Working CI on every push to develop/main.
Three jobs: backend, frontend, python-worker.

**Files to create/modify:**
- `.github/workflows/ci.yml` (SindriStudio root)

See TESTING_STRATEGY.md for the exact job definitions.

**Acceptance:**
- [ ] Push to develop triggers CI
- [ ] All three jobs pass green
- [ ] Failing test causes CI to fail (verify by temporarily adding a broken test)

---

### P1-J2 — Integration smoke test

**Goal:** Manual verification that the full system works end-to-end.
Documents the result in the task report. No automated test (requires real hardware setup).

**Verification sequence:**
1. `cargo build --release`
2. `./target/release/sindristudio` — browser opens to BloomeryUI
3. BloomeryUI shows connected status (green)
4. SystemStats shows real hardware info
5. WorkerList shows one worker as idle
6. Submit a job via JobSubmitForm with `{"nodes":[],"edges":[]}`
7. Job appears in JobList as queued then dispatched
8. EventLog shows job.queued and job.started events
9. System.stats events arrive every 5 seconds
10. Ctrl+C graceful shutdown — no hanging processes

Document each step result in the P1-J2 report.

---

## PHASE 1 EXPLICITLY OUT OF SCOPE

- Real ML inference (Phase 2)
- Artifact storage and image display (Phase 2)
- Node graph canvas (Phase 3)
- LoRA / ControlNet support (Phase 3+)
- Multi-GPU routing (Phase 3)
- Authentication (Phase 4)
- Apple Silicon support (Phase 4)
- Extension/plugin system (Phase 4)
