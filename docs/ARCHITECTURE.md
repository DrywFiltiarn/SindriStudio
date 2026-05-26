# ARCHITECTURE.md — SindriStudio System Architecture
# Canonical definition of every crate, module, and component.
# All file paths created by Cline must match this document.

---

## SYSTEM OVERVIEW

```
┌─────────────────────────────────────────────────────────────────┐
│  sindristudio  (launcher binary)                                │
│  ├── Starts AnvilML HTTP server                                 │
│  ├── Spawns Python inference workers                            │
│  └── Opens browser to BloomeryUI                               │
└─────────────────┬───────────────────────────────────────────────┘
                  │
    ┌─────────────▼──────────────┐
    │  AnvilML (Rust backend)    │◄──── REST + WebSocket ────► BloomeryUI
    │  axum HTTP server :8188    │                             (React SPA)
    │  ├── Scheduler             │
    │  ├── Worker pool           │
    │  ├── Model registry        │
    │  └── Hardware detector     │
    └─────────────┬──────────────┘
                  │ IPC (stdin/stdout, msgpack)
    ┌─────────────▼──────────────┐
    │  Python workers            │
    │  ├── ZiT / SDXL inference  │
    │  ├── Model manager         │
    │  └── Node executor         │
    └────────────────────────────┘
```

AnvilML serves BloomeryUI's static files from `dist/` in production.
In development, BloomeryUI runs on its own Vite dev server (:5173) with proxy.

---

## BACKEND — AnvilML Cargo Workspace

### Workspace layout

```
backend/                              ← AnvilML repo root
├── Cargo.toml                        ← workspace manifest (lists all members)
├── Cargo.lock
├── anvilml.toml.example
├── .gitignore
├── README.md
│
├── crates/
│   ├── anvilml-core/                 ← shared types, config, errors (no deps on other crates)
│   ├── anvilml-hardware/             ← device detection (depends: core)
│   ├── anvilml-registry/             ← model scanning and indexing (depends: core)
│   ├── anvilml-ipc/                  ← IPC message types + framing (depends: core)
│   ├── anvilml-worker/               ← Python process pool + supervisor (depends: core, ipc, hardware)
│   ├── anvilml-scheduler/            ← job queue, DAG, VRAM ledger (depends: core, ipc, worker, registry)
│   ├── anvilml-server/               ← axum HTTP + WebSocket (depends: all above)
│   └── anvilml-openapi/              ← OpenAPI spec generation (depends: server)
│
├── src/
│   └── main.rs                       ← sindristudio launcher binary entry point
│                                       (depends: anvilml-server as lib)
│
├── worker/                           ← Python inference worker (not a Rust crate)
│   ├── requirements.txt
│   ├── worker_main.py
│   ├── ipc.py
│   ├── executor.py
│   ├── model_manager.py
│   ├── nodes/
│   │   ├── __init__.py               ← NODE_REGISTRY dict
│   │   ├── base.py                   ← BaseNode ABC
│   │   └── zit.py                    ← ZiT pipeline nodes (Phase 2)
│   └── tests/
│       ├── conftest.py
│       ├── test_ipc.py
│       ├── test_executor.py
│       └── test_nodes_zit.py
│
├── tests/                            ← Rust integration tests (workspace-level)
│   ├── api_health.rs
│   ├── api_system.rs
│   └── api_jobs.rs
│
└── scripts/
    ├── install_worker_deps.sh        ← Linux venv setup
    ├── install_worker_deps.ps1       ← Windows venv setup
    └── test_inference.py             ← standalone inference test (dev only)
```

### Workspace Cargo.toml

```toml
[workspace]
resolver = "2"
members = [
    "crates/anvilml-core",
    "crates/anvilml-hardware",
    "crates/anvilml-registry",
    "crates/anvilml-ipc",
    "crates/anvilml-worker",
    "crates/anvilml-scheduler",
    "crates/anvilml-server",
    "crates/anvilml-openapi",
    ".",          # launcher binary
]

[workspace.dependencies]
# Pin shared dependency versions here — crates reference with { workspace = true }
tokio       = { version = "1",   features = ["full"] }
axum        = { version = "0.7", features = ["ws", "macros"] }
serde       = { version = "1",   features = ["derive"] }
serde_json  = "1"
anyhow      = "1"
thiserror   = "1"
tracing     = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
uuid        = { version = "1",   features = ["v4", "serde"] }
chrono      = { version = "0.4", features = ["serde"] }
sqlx        = { version = "0.8", features = ["sqlite", "runtime-tokio", "chrono", "uuid"] }
```

---

## CRATE RESPONSIBILITIES

### `anvilml-core`
**Purpose:** Shared types, configuration, and error definitions.
No dependencies on other AnvilML crates. All other crates depend on this.

Key contents:
- `config.rs` — `ServerConfig`, `RocmConfig`, `ModelDirConfig`, `ModelKind` enum
- `error.rs` — `AnvilError` enum (thiserror), `Result<T>` type alias
- `types/job.rs` — `Job`, `JobStatus`, `JobSettings` structs
- `types/model.rs` — `ModelMeta`, `ArtifactMeta` structs
- `types/hardware.rs` — `GpuDevice`, `DeviceType`, `HardwareInfo`, `InferenceCaps` structs
- `types/worker.rs` — `WorkerInfo`, `WorkerStatus` structs

All types that appear in the REST API response bodies live here.
`anvilml-openapi` reads these to generate the OpenAPI spec.

### `anvilml-hardware`
**Purpose:** Device detection across all supported backends.
Detects CUDA, ROCm, IPEX, and CPU. Platform-aware.

Key contents:
- `detector.rs` — `HardwareDetector::detect() -> Result<HardwareInfo>` (async)
- `rocm.rs` — ROCm-specific detection (rocm-smi, rocminfo, gfx arch, ReBAR)
- `cuda.rs` — CUDA detection (nvidia-smi)
- `ipex.rs` — Intel IPEX detection
- `cpu.rs` — CPU fallback (always succeeds)
- `mock.rs` — `MockHardwareDetector` for testing (feature-gated)

Feature flags:
- `real-hardware` (default) — uses actual system calls
- `mock-hardware` — returns configurable fake device data (used in CI)

### `anvilml-registry`
**Purpose:** Scan model directories, maintain index, persist to SQLite.

Key contents:
- `registry.rs` — `ModelRegistry` struct
  - `scan(dirs) -> Result<()>` — walks dirs, upserts to DB
  - `list(filter) -> Result<Vec<ModelMeta>>`
  - `get(id) -> Result<Option<ModelMeta>>`
  - `estimate_vram_mib(path, dtype) -> u64`
- `scanner.rs` — filesystem walk, extension filtering, metadata extraction

### `anvilml-ipc`
**Purpose:** IPC message types and framing shared between Rust and Python.
This crate's types ARE the IPC protocol. See `docs/IPC_PROTOCOL.md`.

Key contents:
- `messages.rs` — `WorkerMessage` enum (Rust→Python), `WorkerEvent` enum (Python→Rust)
  All variants exactly match `docs/IPC_PROTOCOL.md`. Field names are the contract.
- `framing.rs` — `read_message()` and `write_event()` async framing functions
  (4-byte big-endian length prefix + msgpack body)

### `anvilml-worker`
**Purpose:** Python worker process lifecycle management.

Key contents:
- `pool.rs` — `WorkerPool` — spawn, supervise, route messages
- `managed.rs` — `ManagedWorker` — per-process state, tx channel, status
- `env.rs` — environment variable injection per device type
  - `rocm_env(device, config) -> HashMap<String, String>`
  - `cuda_env(device, config) -> HashMap<String, String>`
  - `ipex_env(device, config) -> HashMap<String, String>`
- `ipc_bridge.rs` — async tasks for stdin writer and stdout reader per worker

### `anvilml-scheduler`
**Purpose:** Job queue, dispatch, VRAM ledger, look-ahead prefetch.

Key contents:
- `scheduler.rs` — `JobScheduler` — submit/cancel/list, dispatch loop
- `queue.rs` — `JobQueue` — priority-ordered `VecDeque<Job>`
- `ledger.rs` — `VramLedger` — per-device VRAM budget tracking
- `dag.rs` — DAG validation and topological sort for workflow graphs

### `anvilml-server`
**Purpose:** axum HTTP server, WebSocket broadcaster, route handlers.
This crate is the public face of AnvilML. It is also a **library crate**
so the launcher binary can call `anvilml_server::start(config).await`.

Key contents:
- `lib.rs` — `pub async fn start(config: ServerConfig) -> Result<()>`
- `router.rs` — assembles all routes, middleware (CORS, compression, tracing)
- `state.rs` — `AppState` — Arc-wrapped shared state passed to all handlers
- `handlers/health.rs` — `GET /health`
- `handlers/system.rs` — `GET /v1/system`
- `handlers/jobs.rs` — `POST /v1/jobs`, `GET /v1/jobs`, `GET /v1/jobs/:id`, `DELETE /v1/jobs/:id`
- `handlers/models.rs` — `GET /v1/models`, `GET /v1/models/:id`
- `handlers/workers.rs` — `GET /v1/workers`
- `handlers/artifacts.rs` — `GET /v1/artifacts`, `GET /v1/artifacts/:hash`
- `handlers/events.rs` — `GET /v1/events` (WebSocket upgrade)
- `ws/broadcaster.rs` — `EventBroadcaster` — wraps `broadcast::Sender<WsEvent>`
- `error.rs` — `ApiError` enum → HTTP responses with `{"error":{"code","message"}}`

### `anvilml-openapi`
**Purpose:** Generate OpenAPI 3.1 JSON spec from the server's routes and types.
Output is written to `backend/openapi.json` at build time.
BloomeryUI's `pnpm generate-types` reads this file.

Key contents:
- `generate.rs` — walks `anvilml-server` routes, extracts type info, writes JSON
- `bin/generate_openapi.rs` — binary that runs generation and exits

### Launcher binary (`backend/src/main.rs`)
**Purpose:** `sindristudio` / `sindristudio.exe` — the single user-facing binary.

Responsibilities:
1. Parse CLI args (`--config`, `--port`, `--no-browser`)
2. Load `ServerConfig`
3. Call `anvilml_server::start(config)` which starts everything
4. On startup: open default browser to `http://127.0.0.1:{port}`
5. Block until Ctrl+C / SIGTERM, then graceful shutdown

Browser opening: use the `open` crate (cross-platform, works on Windows + Linux).

---

## FRONTEND — BloomeryUI

### Repository layout

```
frontend/                             ← BloomeryUI repo root
├── package.json
├── pnpm-lock.yaml
├── tsconfig.json
├── tsconfig.node.json
├── vite.config.ts
├── tailwind.config.ts
├── postcss.config.js
├── .eslintrc.cjs
├── .gitignore
├── README.md
├── index.html
│
└── src/
    ├── main.tsx
    ├── App.tsx
    ├── styles/
    │   └── globals.css
    │
    ├── generated/                    ← AUTO-GENERATED — never hand-edit
    │   └── api-types.ts              ← from backend/openapi.json via openapi-typescript
    │
    ├── api/
    │   ├── client.ts                 ← base fetch wrapper, error handling
    │   ├── websocket.ts              ← WebSocketManager
    │   └── endpoints/
    │       ├── system.ts
    │       ├── jobs.ts
    │       ├── models.ts
    │       ├── workers.ts
    │       └── artifacts.ts
    │
    ├── stores/
    │   ├── connection.store.ts
    │   ├── system.store.ts
    │   ├── jobs.store.ts
    │   ├── events.store.ts
    │   └── artifacts.store.ts
    │
    ├── hooks/
    │   ├── useConnection.ts
    │   ├── useSystemStats.ts
    │   ├── useJobEvents.ts
    │   └── useArtifacts.ts
    │
    ├── components/
    │   ├── layout/
    │   │   ├── AppShell.tsx
    │   │   ├── Sidebar.tsx
    │   │   ├── TopBar.tsx
    │   │   └── index.ts
    │   ├── connection/
    │   │   ├── ConnectionPanel.tsx
    │   │   ├── StatusBadge.tsx
    │   │   └── index.ts
    │   ├── system/
    │   │   ├── SystemStats.tsx
    │   │   ├── VramBar.tsx
    │   │   ├── WorkerList.tsx
    │   │   └── index.ts
    │   ├── jobs/
    │   │   ├── ZitJobForm.tsx        ← Phase 2: structured ZiT form
    │   │   ├── JobList.tsx
    │   │   ├── JobDetail.tsx
    │   │   ├── JobProgress.tsx
    │   │   └── index.ts
    │   ├── models/
    │   │   ├── ModelList.tsx
    │   │   └── index.ts
    │   ├── artifacts/
    │   │   ├── ArtifactGallery.tsx
    │   │   ├── ArtifactCard.tsx
    │   │   └── index.ts
    │   └── events/
    │       ├── EventLog.tsx
    │       ├── EventEntry.tsx
    │       └── index.ts
    │
    ├── pages/
    │   ├── TestPanel.tsx             ← Phase 1: developer test panel
    │   └── NotFound.tsx
    │
    └── tests/
        ├── setup.ts
        ├── api/
        │   └── client.test.ts
        ├── stores/
        │   ├── connection.store.test.ts
        │   └── jobs.store.test.ts
        └── components/
            ├── StatusBadge.test.tsx
            └── VramBar.test.tsx
```

### Package dependencies

```json
{
  "dependencies": {
    "react": "^18.3.0",
    "react-dom": "^18.3.0",
    "zustand": "^4.5.0",
    "clsx": "^2.1.0"
  },
  "devDependencies": {
    "@types/react": "^18.3.0",
    "@types/react-dom": "^18.3.0",
    "@vitejs/plugin-react": "^4.3.0",
    "typescript": "^5.4.0",
    "vite": "^5.3.0",
    "vitest": "^1.6.0",
    "@testing-library/react": "^16.0.0",
    "@testing-library/user-event": "^14.5.0",
    "jsdom": "^24.0.0",
    "tailwindcss": "^3.4.0",
    "autoprefixer": "^10.4.0",
    "postcss": "^8.4.0",
    "eslint": "^8.57.0",
    "@typescript-eslint/eslint-plugin": "^7.0.0",
    "@typescript-eslint/parser": "^7.0.0",
    "eslint-plugin-react-hooks": "^4.6.0",
    "openapi-typescript": "^7.0.0"
  },
  "scripts": {
    "dev":            "vite",
    "build":          "tsc && vite build",
    "preview":        "vite preview",
    "test":           "vitest",
    "test:run":       "vitest run",
    "type-check":     "tsc --noEmit",
    "lint":           "eslint src --ext ts,tsx",
    "generate-types": "openapi-typescript ../backend/openapi.json -o src/generated/api-types.ts"
  }
}
```

### Design tokens (tailwind.config.ts)

```typescript
colors: {
  surface: {
    DEFAULT: '#0d0f12',
    raised:  '#13161b',
    overlay: '#1a1e25',
  },
  border: {
    DEFAULT: '#262c36',
    strong:  '#2f3847',
  },
  accent: {
    green:  '#4af0a0',
    blue:   '#3dd6e0',
    amber:  '#f0a04a',
    red:    '#f04a6e',
  },
  text: {
    primary: '#d4dde8',
    muted:   '#6b7a8d',
    dim:     '#3f4f62',
  },
}
```

---

## DEPENDENCY GRAPH

```
anvilml-core
    ↑
    ├── anvilml-hardware
    ├── anvilml-registry
    ├── anvilml-ipc
    │       ↑
    │   anvilml-worker ← anvilml-hardware
    │       ↑
    │   anvilml-scheduler ← anvilml-registry
    │       ↑
    │   anvilml-server (lib)
    │       ↑
    │   launcher binary (backend/src/main.rs)
    │       ↑
    │   anvilml-openapi (build tool, not linked at runtime)
```

---

## OPENAPI TYPE SHARING

AnvilML is the source of truth for all API types.
BloomeryUI consumes generated types — never writes its own API type definitions.

Flow:
1. `anvilml-openapi` generates `backend/openapi.json` (run: `cargo run -p anvilml-openapi`)
2. BloomeryUI runs `pnpm generate-types` which reads `../backend/openapi.json`
   and writes `src/generated/api-types.ts` via `openapi-typescript`
3. All frontend API code imports from `src/generated/api-types.ts`
4. `src/generated/` is committed to the BloomeryUI repo so CI doesn't need the backend

In CI: `openapi.json` is committed alongside the code that generates it.
Type drift is caught by: if `openapi.json` changes, `generate-types` must be re-run,
and TypeScript compile errors will surface any shape mismatches.

See `docs/OPENAPI_GENERATION.md` for full details.
