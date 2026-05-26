# ENVIRONMENT.md — SindriStudio System Configuration
# Read at the start of every session. These are ground truths.
# Never hardcode values that contradict this file.
# Last updated: 2026

---

## REPOSITORY STRUCTURE

```
SindriStudio/                        ← root repo (github.com/DrywFiltiarn/SindriStudio)
├── .clinerules                      ← Cline session rules (governs everything)
├── .cline/
│   ├── state/
│   │   └── CURRENT_TASK.md         ← active task state (single source of truth)
│   └── reports/
│       └── {TASK-ID}.md            ← one report per completed task
├── docs/                            ← all specification documents
│   ├── ENVIRONMENT.md               ← this file
│   ├── ARCHITECTURE.md              ← workspace layout, crate map, component map
│   ├── API_CONTRACT.md              ← REST + WebSocket API (OpenAPI-first)
│   ├── IPC_PROTOCOL.md              ← Rust↔Python msgpack protocol
│   ├── OPENAPI_GENERATION.md        ← how OpenAPI→TypeScript types are generated
│   ├── TESTING_STRATEGY.md          ← mock boundaries, CI vs local, coverage rules
│   ├── TASKS_PHASE1.md              ← atomic task list, Phase 1
│   ├── TASKS_PHASE2.md              ← atomic task list, Phase 2
│   ├── TASKS_PHASE3.md              ← milestone definitions, Phase 3
│   └── TASKS_PHASE4.md              ← milestone definitions, Phase 4
├── models/                          ← shared model storage (all platforms)
│   ├── clip/
│   ├── diffusion/
│   ├── vae/
│   ├── lora/
│   ├── controlnet/
│   ├── unet/
│   └── upscale/
├── backend/                         ← AnvilML submodule (github.com/DrywFiltiarn/AnvilML)
└── frontend/                        ← BloomeryUI submodule (github.com/DrywFiltiarn/BloomeryUI)
```

---

## PLATFORM TARGETS

SindriStudio supports two platforms fully from day one.
Apple Silicon is planned for a later phase and must not be blocked by early decisions.

| Platform       | Status      | Notes                                     |
|----------------|-------------|-------------------------------------------|
| Linux x86_64   | Supported   | Primary development target                |
| Windows 10/11  | Supported   | Full parity with Linux                    |
| macOS ARM64    | Planned     | Apple Silicon — later phase               |

Platform-conditional code uses `#[cfg(target_os = "windows")]` / `#[cfg(unix)]`.
Never assume Unix-only APIs (e.g. Unix domain sockets). Use named pipes on Windows.

---

## HARDWARE SUPPORT MATRIX

| Accelerator     | Linux   | Windows  | Backend                              |
|-----------------|---------|----------|--------------------------------------|
| NVIDIA GPU      | ✓       | ✓        | PyTorch CUDA                         |
| AMD GPU         | ✓       | ✓        | PyTorch ROCm (via pip wheel)         |
| Intel GPU (Arc) | ✓       | ✓        | PyTorch IPEX                         |
| CPU             | ✓       | ✓        | PyTorch CPU (float32, quantized)     |
| Apple Silicon   | —       | —        | Planned: PyTorch MPS                 |

### ROCm on Windows
ROCm 7.2+ ships complete Windows support via pip wheels.
No external ROCm installation required — the wheel contains everything.
HSA/HIP runtime is bundled in the wheel.
`pip install torch --index-url https://download.pytorch.org/whl/rocm7.2`

### Device Selection Strategy
The Rust hardware detector enumerates all devices at startup.
The Python worker receives its device assignment via environment variable.
Device selection priority (unless overridden in config): ROCm > CUDA > IPEX > CPU.

---

## DEVELOPER MACHINE (primary test target)

| Property         | Value                                          |
|------------------|------------------------------------------------|
| CPU              | Intel Xeon E5-2680 v4 (14C / 28T, single socket) |
| GPU              | AMD RDNA4 (gfx1201)                            |
| ROCm version     | 7.x nightly                                    |
| OS               | Linux                                          |
| System RAM       | ≥ 64 GiB (assumed)                             |
| GPU VRAM         | ≥ 16 GiB (assumed)                             |
| ReBAR            | Enabled                                        |

### Threading — Xeon E5-2680 v4
One socket, no NUMA split. Use all threads.

| Parameter              | Value | Rationale                                |
|------------------------|-------|------------------------------------------|
| `torch.set_num_threads`| 14    | Physical cores — avoids HT contention    |
| `torch.set_num_interop_threads` | 4 | Parallel independent graph ops      |
| `OMP_NUM_THREADS`      | 14    | OpenMP alignment                         |
| `MKL_NUM_THREADS`      | 14    | MKL alignment                            |
| Tokio worker threads   | 28    | Full logical thread count                |

Thread config must happen BEFORE importing torch. Set env vars first, then import.

---

## PYTHON ENVIRONMENT

| Property        | Value                                      |
|-----------------|--------------------------------------------|
| Python version  | 3.12                                       |
| Venv location   | `backend/.venv/`                           |
| Creation        | `python3.12 -m venv backend/.venv`         |
| Python binary   | `backend/.venv/bin/python` (Linux)         |
|                 | `backend/.venv/Scripts/python.exe` (Win)   |
| Requirements    | `backend/worker/requirements.txt`          |

---

## ROCm ENVIRONMENT VARIABLES
Injected by Rust supervisor into Python worker process environment.
Python worker never sets these itself.

```
HIP_VISIBLE_DEVICES={device_index}
ROCM_HOME=/opt/rocm               # Linux default; Windows: determined at runtime
ROCM_PATH=/opt/rocm
ROCBLAS_USE_HIPBLASLT=1
HIPBLASLT_LOG_MASK=0
AOTRITON_ENABLE_EXPERIMENTAL=1
PYTORCH_TUNABLEOP_ENABLED=0        # off by default; enable post-MVP
HSA_NO_SCRATCH_RECLAIM=1
MIOPEN_FIND_MODE=3
# Conditionally set only if needed:
HSA_OVERRIDE_GFX_VERSION=12.0.1
```

---

## NETWORK

| Property          | Value                      |
|-------------------|----------------------------|
| Backend host      | `127.0.0.1`                |
| Backend port      | `8188`                     |
| Frontend dev port | `5173`                     |
| WebSocket path    | `/v1/events`               |
| CORS in dev       | Allow all origins          |
| API prefix        | `/v1/`                     |

---

## MODEL DIRECTORY LAYOUT

Relative to SindriStudio root:

```
models/
├── clip/           CLIP text encoders
├── diffusion/      Main diffusion model checkpoints
├── vae/            VAE models
├── lora/           LoRA adapter weights
├── controlnet/     ControlNet models
├── unet/           Standalone UNet checkpoints
└── upscale/        Upscaling models
```

Backend config references these as paths relative to the config file location.
Never hardcode absolute paths — always resolve from config at runtime.

---

## TOOLCHAIN

| Tool       | Version         | Check                   |
|------------|-----------------|-------------------------|
| Rust       | stable latest   | `rustc --version`       |
| Cargo      | with Rust       | `cargo --version`       |
| Python     | 3.12.x          | `python3.12 --version`  |
| Node.js    | 20 LTS          | `node --version`        |
| pnpm       | 9.x             | `pnpm --version`        |
| git        | 2.x             | `git --version`         |

---

## RUNTIME CONFIG FILE

AnvilML reads `backend/anvilml.toml` at startup.
`backend/anvilml.toml` is gitignored (contains local paths).
`backend/anvilml.toml.example` is committed as the template.

```toml
host = "127.0.0.1"
port = 8188
database_url = "sqlite://./anvilml.db"
artifact_dir = "../artifacts"
python_bin = ".venv/bin/python"           # Windows: ".venv/Scripts/python.exe"
worker_script = "worker/worker_main.py"
max_workers_per_device = 1
vram_budget_fraction = 0.90
open_browser = true                       # launcher opens browser on start

[hardware]
# Leave empty for auto-detection. Override only if detection is wrong.
# force_device_type = "rocm"   # rocm | cuda | ipex | cpu

[rocm]
target_gfx = "gfx1201"
force_hipblaslt = true
flash_attention = true
# override_gfx_version = "12.0.1"   # uncomment if needed for nightly ROCm

[[model_dirs]]
path = "../models/clip"
kind = "clip"

[[model_dirs]]
path = "../models/diffusion"
kind = "diffusion"

[[model_dirs]]
path = "../models/vae"
kind = "vae"

[[model_dirs]]
path = "../models/lora"
kind = "lora"

[[model_dirs]]
path = "../models/controlnet"
kind = "controlnet"

[[model_dirs]]
path = "../models/unet"
kind = "unet"

[[model_dirs]]
path = "../models/upscale"
kind = "upscale"
```
