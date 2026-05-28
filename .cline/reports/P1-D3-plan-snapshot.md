# P1-D3: anvilml-worker — Environment Variable Injection

## STEP 1 — PLAN ONLY (No Implementation)

---

## 1. Task Overview

**From TASKS_PHASE1.md:**

> **P1-D3** — `anvilml-worker`: ROCm/CUDA/IPEX env injection
> 
> Prerequisites: P1-D1 (`WorkerPool` + spawn)
>
> **Goal:** Implement per-device environment variable injection. Each device type (ROCm, CUDA, IPEX) gets the correct env vars injected into the Python worker process before spawn.
>
> **Files to create:** `crates/anvilml-worker/src/env.rs`
>
> **Implement:**
> ```rust
> pub fn build_worker_env(
>     device: &GpuDevice,
>     config: &ServerConfig,
> ) -> HashMap<String, String>
> ```
> Returns the full environment map for the worker process.
> See ENVIRONMENT.md for the exact variables per device type.
>
> **Platform-aware:** On Windows, Python binary path uses backslashes,
> `HIP_PATH` points to the ROCm wheel location, not `/opt/rocm`.

**Acceptance criteria (from task spec):**
- [ ] `cargo test -p anvilml-worker --features mock-hardware` exits 0

---

## 2. Existing Context Analysis

### Current `lib.rs` structure
```
crates/anvilml-worker/src/
├── lib.rs          # pub mod ipc_bridge; pub mod managed; pub mod pool;
├── pool.rs         # WorkerPool — spawn_for_device() inline env setup
├── managed.rs      # ManagedWorker — state machine
└── ipc_bridge.rs   # stdin/stdout reader tasks
```

### Current `pool.rs` environment handling (to be replaced)

In `spawn_for_device()` (line 53–68), env vars are set **inline** on the `Command`:
```rust
cmd.arg(&worker_script)
    .env("ANVILML_WORKER_MOCK", "1")
    .env("ANVILML_DEVICE_TYPE", device_type_str(&device_type))
    .env("ANVILML_DEVICE_INDEX", device_index.to_string())
    // ... PYTHONPATH, current_dir, stdout/stderr
```

The **respawn closure** (line 124–138) duplicates this same inline pattern.

### Problems with current approach:
1. **Duplication**: Same env setup duplicated in spawn + respawn paths
2. **Incomplete**: Only sets `ANVILML_WORKER_MOCK`, `ANVILML_DEVICE_TYPE`, `ANVILML_DEVICE_INDEX` — no ROCm/CUDA/IPEX-specific vars (HIP_VISIBLE_DEVICES, CUDA_VISIBLE_DEVICES, etc.)
3. **No platform awareness**: Hardcoded `/opt/rocm` paths not considered
4. **No config integration**: `RocmConfig` fields (`force_hipblaslt`, `flash_attention`, `override_gfx_version`) are never used in env injection
5. **Hard to test**: Inline `.env()` calls make it impossible to assert on the full env map

### What `ServerConfig` provides (from `config.rs`):
- `python_bin: PathBuf` — path to Python executable
- `worker_script: PathBuf` — path to worker_main.py
- `max_workers_per_device: usize`
- `vram_budget_fraction: f64`
- `rocm: RocmConfig` — `target_gfx`, `force_hipblaslt`, `flash_attention`, `override_gfx_version`
- `hardware: HardwareOverrideConfig` — `force_device_type`

### What `GpuDevice` provides (from `hardware.rs`):
- `index: u32` — device index for visible devices env vars
- `device_type: DeviceType` — Rocm/Cuda/Ipx/Cpu enum
- `gfx_arch: Option<String>` — e.g. "gfx1201"
- `vram_total_mib: u64`
- `capabilities: InferenceCaps` — fp16, bf16, fp8, flash_attention, hipblaslt

### What `ManagedWorker` tracks (from `managed.rs`):
- `device_index`, `device_type` — already available per-worker
- No env state currently tracked

---

## 3. Proposed `env.rs` Module Design

### File: `crates/anvilml-worker/src/env.rs`

```rust
//! Per-device environment variable injection for Python worker processes.
//!
//! Produces a complete `HashMap<String, String>` of environment variables
//! to set on the subprocess before spawn. Called by `WorkerPool::spawn_for_device()`
//! and the respawn closure in `ipc_bridge`.
```

### Module structure:

```env.rs├── build_worker_env(device: &GpuDevice, config: &ServerConfig) -> HashMap<String, String>
    │   // Entry point — delegates to base + device-type-specific helpers
    │
    ├── build_base_env(config: &ServerConfig, device_index: u32) -> HashMap<String, String>
    │   // Common vars for ALL device types:
    │   // - ANVILML_WORKER_MOCK (if set in env)
    │   // - OMP_NUM_THREADS
    │   // - MKL_NUM_THREADS
    │   // - PYTHONPATH (sandbox root)
    │   // - Torch threading hints
    │
    ├── build_rocm_env(device: &GpuDevice, config: &ServerConfig) -> HashMap<String, String>
    │   // ROCm-specific vars from ENVIRONMENT.md:
    │   // - HIP_VISIBLE_DEVICES={index}
    │   // - ROCM_HOME (platform-aware)
    │   // - ROCM_PATH
    │   // - ROCBLAS_USE_HIPBLASLT (from config.force_hipblaslt)
    │   // - HIPBLASLT_LOG_MASK=0
    │   // - AOTRITON_ENABLE_EXPERIMENTAL=1
    │   // - PYTORCH_TUNABLEOP_ENABLED=0
    │   // - HSA_NO_SCRATCH_RECLAIM=1
    │   // - MIOPEN_FIND_MODE=3
    │   // - HSA_OVERRIDE_GFX_VERSION (from config.rocm.override_gfx_version)
    │
    ├── build_cuda_env(device: &GpuDevice) -> HashMap<String, String>
    │   // CUDA-specific:
    │   // - CUDA_VISIBLE_DEVICES={index}
    │
    ├── build_ipex_env(device: &GpuDevice) -> HashMap<String, String>
    │   // IPEX-specific:
    │   // - IPEX_VISIBLE_DEVICES={index}  (or whatever Intel uses)
    │
    └── build_cpu_env() -> HashMap<String, String>
        // CPU-specific (minimal):
        // - No GPU-visible-devices vars
        // - May set TORCH_CPU_AFFINITY or similar if needed later
```

### Design decisions:

1. **`build_worker_env` is a pure function** — takes references, returns owned `HashMap`. No I/O, no async, no side effects. This makes it trivially testable.

2. **Helper functions are also pure** — each device-type helper returns its own `HashMap<String, String>` which gets merged into the base env in `build_worker_env`.

3. **Base env includes threading vars from ENVIRONMENT.md** (Xeon E5-2680 v4 profile):
   - `OMP_NUM_THREADS=14`
   - `MKL_NUM_THREADS=14`
   - These are set BEFORE importing torch, as specified in ENVIRONMENT.md

4. **Platform-aware ROCm paths**: Uses `#[cfg(target_os = "windows")]` to differentiate:
   - Linux: `ROCM_HOME=/opt/rocm`, `ROCM_PATH=/opt/rocm`
   - Windows: `HIP_PATH` points to the ROCm wheel's internal HIP runtime location (determined at runtime via config or well-known pip wheel path)

5. **Config-driven overrides**: `RocmConfig` fields control conditional env vars:
   - `force_hipblaslt=true` → `ROCBLAS_USE_HIPBLASLT=1`
   - `override_gfx_version=Some("12.0.1")` → `HSA_OVERRIDE_GFX_VERSION=12.0.1`
   - `flash_attention=true` → `AOTRITON_ENABLE_EXPERIMENTAL=1`

6. **`ANVILML_WORKER_MOCK` is NOT set by `build_worker_env`** — it's an external test harness concern, passed via the caller's existing `.env()` call. The env module focuses on *device-specific* runtime configuration.

---

## 4. API Signatures

### Primary entry point:
```rust
/// Build the full environment map for a Python worker process bound to `device`.
///
/// Merges base threading/env vars with device-type-specific variables.
/// Does NOT set ANVILML_WORKER_MOCK — that is caller-controlled.
pub fn build_worker_env(
    device: &GpuDevice,
    config: &ServerConfig,
) -> HashMap<String, String>
```

### Per-device-type helpers (all pure, testable independently):
```rust
/// Base environment variables common to all device types.
pub fn build_base_env(
    config: &ServerConfig,
    device_index: u32,
) -> HashMap<String, String>;

/// ROCm-specific environment variables.
pub fn build_rocm_env(
    device: &GpuDevice,
    config: &ServerConfig,
) -> HashMap<String, String>;

/// CUDA-specific environment variables.
pub fn build_cuda_env(
    device: &GpuDevice,
) -> HashMap<String, String>;

/// IPEX-specific environment variables.
pub fn build_ipex_env(
    device: &GpuDevice,
) -> HashMap<String, String>;

/// CPU-specific environment variables (minimal).
pub fn build_cpu_env() -> HashMap<String, String>;
```

### Internal helper:
```rust
/// Merge two env maps; right-hand side takes precedence on key conflicts.
fn merge_env(base: HashMap<String, String>, extra: HashMap<String, String>) -> HashMap<String, String>
```

---

## 5. Environment Variable Mapping Per Device Type

### ROCm (from ENVIRONMENT.md §ROCm ENVIRONMENT VARIABLES)

| Variable | Value | Source | Conditional |
|----------|-------|--------|-------------|
| `HIP_VISIBLE_DEVICES` | `{device.index}` | Always | — |
| `ROCM_HOME` | `/opt/rocm` | Linux default | `#[cfg(unix)]` |
| `ROCM_PATH` | `/opt/rocm` | Linux default | `#[cfg(unix)]` |
| `HIP_PATH` | Windows ROCm wheel path | Platform-aware | `#[cfg(target_os = "windows")]` |
| `ROCBLAS_USE_HIPBLASLT` | `1` | Config | `config.rocm.force_hipblaslt` |
| `HIPBLASLT_LOG_MASK` | `0` | Always | — |
| `AOTRITON_ENABLE_EXPERIMENTAL` | `1` | Config | `config.rocm.flash_attention` |
| `PYTORCH_TUNABLEOP_ENABLED` | `0` | Always (off by default) | — |
| `HSA_NO_SCRATCH_RECLAIM` | `1` | Always | — |
| `MIOPEN_FIND_MODE` | `3` | Always | — |
| `HSA_OVERRIDE_GFX_VERSION` | `{config.rocm.override_gfx_version}` | Config | Only if `Some`

### CUDA (from ENVIRONMENT.md + standard PyTorch conventions)

| Variable | Value | Source |
|----------|-------|--------|
| `CUDA_VISIBLE_DEVICES` | `{device.index}` | Always |

### IPEX (Intel oneAPI Extension for PyTorch)

| Variable | Value | Source |
|----------|-------|--------|
| `ONEAPI_DEVICE_SELECTOR` | `level_zero:` or `gpu` | Intel convention |

**Note:** ENVIRONMENT.md doesn't specify exact IPEX env vars. The implementation will use the standard Intel oneAPI device selector pattern. If Intel uses a different variable, it can be adjusted — the helper function is isolated and easy to change.

### CPU (no GPU-specific vars)

| Variable | Value | Source |
|----------|-------|--------|
| *(none)* | — | No GPU env vars for CPU |

**Note:** CPU workers don't need device-selection env vars. They run on the host CPU.

### Base/Threading variables (all device types)

From ENVIRONMENT.md §DEVELOPER MACHINE threading config:

| Variable | Value | Rationale |
|----------|-------|-----------|
| `OMP_NUM_THREADS` | `14` | Physical cores — avoids HT contention |
| `MKL_NUM_THREADS` | `14` | MKL alignment |

These are set **before** importing torch, as specified in ENVIRONMENT.md.

---

## 6. Platform-Aware Considerations

### ROCm paths

```rust
#[cfg(unix)]
fn rocm_home() -> String {
    "/opt/rocm".to_string()
}

#[cfg(target_os = "windows")]
fn rocm_home() -> String {
    // On Windows, ROCm 7.2+ ships as a pip wheel.
    // The HIP runtime is bundled inside the wheel package.
    // We look for it in the Python site-packages or use a well-known path.
    // For Phase 1, default to the expected pip wheel location:
    std::env::var("HIP_PATH")
        .ok()
        .unwrap_or_else(|| {
            // Typical: <venv>/Lib/site-packages/rocm*/hip/lib/../..
            // But this is fragile — better to document that Windows users
            // should set HIP_PATH in their anvilml.toml or shell.
            "C:/Program Files/AMD/HIP/bin".to_string()
        })
}
```

### Python binary path

Already handled by `config.python_bin` — the existing code passes this to `Command::new()`. No change needed here; `build_worker_env` doesn't touch the Python binary path.

### PYTHONPATH

The existing code sets `PYTHONPATH` inline in `pool.rs` (line 63–66). This should remain as a caller-side concern since it depends on the sandbox root discovery logic (which is pool-specific). The base env helper will NOT set PYTHONPATH — it stays in the spawn code.

### Summary of platform boundaries:

| Concern | Platform | Implementation |
|---------|----------|---------------|
| ROCm paths (`ROCM_HOME`, `ROCM_PATH`) | Linux | Hardcoded `/opt/rocm` |
| ROCm paths | Windows | Runtime lookup + fallback |
| Python binary path | Both | Config-driven (unchanged) |
| PYTHONPATH construction | Both | Pool-side (unchanged) |
| Device index env vars | Both | Platform-independent (`HIP_VISIBLE_DEVICES`, `CUDA_VISIBLE_DEVICES`) |

---

## 7. Integration Point: `pool.rs` `spawn_for_device()`

### Current state:
```rust
// pool.rs lines 53–68 — inline env setup in spawn_for_device()
cmd.arg(&worker_script)
    .env("ANVILML_WORKER_MOCK", "1")
    .env("ANVILML_DEVICE_TYPE", device_type_str(&device_type))
    .env("ANVILML_DEVICE_INDEX", device_index.to_string())
    .current_dir(&sandbox_root)
    .env("PYTHONPATH", format!(...))
```

### After P1-D3 implementation:
```rust
// pool.rs — spawn_for_device() simplified
cmd.arg(&worker_script)
    .env("ANVILML_WORKER_MOCK", "1")          // caller-controlled, stays inline
    .envs(env::build_worker_env(device, &self.config))  // ← NEW: full env map
    .current_dir(&sandbox_root)
    .env("PYTHONPATH", format!(...));          // pool-specific, stays inline
```

### Same pattern in the respawn closure (line 124–138):
```rust
// The respawn closure will also use env::build_worker_env() instead of
// duplicating the inline .env() calls. This eliminates code duplication.
cmd.arg(&worker_script)
    .arg("--worker-id")
    .arg(new_worker_id.to_string())
    .arg("--device-index")
    .arg(d.index.to_string())
    .env("ANVILML_WORKER_MOCK", "1")
    .envs(env::build_worker_env(&d, &self.config))  // ← NEW: unified env
    .current_dir(&sandbox_root)
    .env("PYTHONPATH", format!(...));
```

### What stays in `pool.rs` (not moved to env module):
- `ANVILML_WORKER_MOCK` — test harness control, caller responsibility
- `ANVILML_DEVICE_TYPE`, `ANVILML_DEVICE_INDEX` — these are legacy vars read by the Python worker; they can either stay inline or be absorbed into `build_worker_env`. **Decision: keep them in pool.rs** for backward compatibility with the existing Python worker code (P1-C2 reads `ANVILML_DEVICE_TYPE` and `ANVILML_DEVICE_INDEX`).
- `PYTHONPATH` construction — depends on sandbox root discovery
- `current_dir()` — pool-specific path logic

### What moves to `env.rs`:
- All ROCm/CUDA/IPEX device-selection env vars
- Threading env vars (OMP_NUM_THREADS, MKL_NUM_THREADS)
- Config-driven ROCm tuning vars

---

## 8. Test Plan: 5+ Unit Tests

All tests are pure function calls — no I/O, no async, no subprocess spawning.

### Test 1: `test_rocm_env_includes_hip_visible_devices`
**Description:** Verify that `build_worker_env` with a ROCm device includes `HIP_VISIBLE_DEVICES` set to the device's index.
**Setup:** Create a mock `GpuDevice` with `device_type = Rocm`, `index = 2`. Create default `ServerConfig`.
**Assertion:**
```rust
let env = build_worker_env(&rocm_device, &config);
assert_eq!(env.get("HIP_VISIBLE_DEVICES"), Some(&"2".to_string()));
```
**Device type:** ROCm

### Test 2: `test_rocm_env_respects_force_hipblaslt_config`
**Description:** Verify that `ROCBLAS_USE_HIPBLASLT=1` is set when `config.rocm.force_hipblaslt = true`, and absent when `false`.
**Setup:** Two configs — one with `force_hipblaslt = true`, one with `force_hipblaslt = false`. Same ROCm device.
**Assertion:**
```rust
let env_on = build_worker_env(&device, &config_with_hipblaslt_true);
assert_eq!(env_on.get("ROCBLAS_USE_HIPBLASLT"), Some(&"1".to_string()));

let env_off = build_worker_env(&device, &config_with_hipblaslt_false);
assert_eq!(env_off.get("ROCBLAS_USE_HIPBLASLT"), None);
```
**Device type:** ROCm

### Test 3: `test_cuda_env_includes_cuda_visible_devices`
**Description:** Verify that a CUDA device gets `CUDA_VISIBLE_DEVICES` set to its index.
**Setup:** Create a mock `GpuDevice` with `device_type = Cuda`, `index = 0`. Default config.
**Assertion:**
```rust
let env = build_worker_env(&cuda_device, &config);
assert_eq!(env.get("CUDA_VISIBLE_DEVICES"), Some(&"0".to_string()));
// ROCm vars must NOT be present
assert_eq!(env.get("HIP_VISIBLE_DEVICES"), None);
```
**Device type:** CUDA

### Test 4: `test_cpu_env_excludes_gpu_device_vars`
**Description:** Verify that a CPU device does NOT get any GPU-specific env vars (`HIP_VISIBLE_DEVICES`, `CUDA_VISIBLE_DEVICES`, etc.).
**Setup:** Create a mock `GpuDevice` with `device_type = Cpu`, `index = 0`. Default config.
**Assertion:**
```rust
let env = build_worker_env(&cpu_device, &config);
assert_eq!(env.get("HIP_VISIBLE_DEVICES"), None);
assert_eq!(env.get("CUDA_VISIBLE_DEVICES"), None);
// Base threading vars SHOULD still be present
assert_eq!(env.get("OMP_NUM_THREADS"), Some(&"14".to_string()));
```
**Device type:** CPU

### Test 5: `test_rocm_env_override_gfx_version_conditionally_set`
**Description:** Verify that `HSA_OVERRIDE_GFX_VERSION` is only set when `config.rocm.override_gfx_version` is `Some(...)`.
**Setup:** Two configs — one with `override_gfx_version = Some("12.0.1")`, one with `None`.
**Assertion:**
```rust
let env_with_override = build_worker_env(&device, &config_with_override);
assert_eq!(env_with_override.get("HSA_OVERRIDE_GFX_VERSION"), Some(&"12.0.1".to_string()));

let env_no_override = build_worker_env(&device, &config_without_override);
assert_eq!(env_no_override.get("HSA_OVERRIDE_GFX_VERSION"), None);
```
**Device type:** ROCm

### Test 6: `test_base_env_includes_threading_vars`
**Description:** Verify that the base env includes `OMP_NUM_THREADS=14` and `MKL_NUM_THREADS=14` for all device types.
**Setup:** Call `build_worker_env` with each device type (Rocm, Cuda, Ipx, Cpu). Same device index.
**Assertion:**
```rust
for device_type in [DeviceType::Rocm, DeviceType::Cuda, DeviceType::Ipx, DeviceType::Cpu] {
    let env = build_worker_env(&make_device(device_type, 0), &config);
    assert_eq!(env.get("OMP_NUM_THREADS"), Some(&"14".to_string()));
    assert_eq!(env.get("MKL_NUM_THREADS"), Some(&"14".to_string()));
}
```
**Device type:** All

### Test 7: `test_env_does_not_include_anvilml_worker_mock`
**Description:** Verify that `build_worker_env` does NOT set `ANVILML_WORKER_MOCK` — that is caller-controlled.
**Setup:** Call `build_worker_env` with any device/config combination.
**Assertion:**
```rust
let env = build_worker_env(&rocm_device, &config);
assert_eq!(env.get("ANVILML_WORKER_MOCK"), None);
```
**Device type:** ROCm (any)

### Test 8: `test_rocm_env_flash_attention_sets_aotriton`
**Description:** Verify that `AOTRITON_ENABLE_EXPERIMENTAL=1` is set when `config.rocm.flash_attention = true`.
**Setup:** Config with `flash_attention = true` vs `false`.
**Assertion:**
```rust
let env_on = build_worker_env(&device, &config_with_flash_attn);
assert_eq!(env_on.get("AOTRITON_ENABLE_EXPERIMENTAL"), Some(&"1".to_string()));

let env_off = build_worker_env(&device, &config_without_flash_attn);
assert_eq!(env_off.get("AOTRITON_ENABLE_EXPERIMENTAL"), None);
```
**Device type:** ROCm

---

## 9. Acceptance Criteria Checklist

### From TASKS_PHASE1.md spec:
- [ ] `crates/anvilml-worker/src/env.rs` created with `build_worker_env()` function
- [ ] `build_worker_env(device: &GpuDevice, config: &ServerConfig) -> HashMap<String, String>` implemented
- [ ] ROCm env includes `HIP_VISIBLE_DEVICES="{index}"`
- [ ] ROCm env includes `ROCBLAS_USE_HIPBLASLT=1` when `force_hipblaslt=true`
- [ ] CUDA env includes `CUDA_VISIBLE_DEVICES="{index}"`
- [ ] CPU env does NOT include `HIP_VISIBLE_DEVICES`
- [ ] Env does NOT include `ANVILML_WORKER_MOCK`
- [ ] Platform-aware ROCm paths (Windows vs Linux)
- [ ] `cargo test -p anvilml-worker --features mock-hardware` exits 0

### Additional acceptance criteria:
- [ ] `env.rs` is declared as `pub mod env;` in `lib.rs`
- [ ] All device-type helper functions are public (for testability)
- [ ] `pool.rs` updated to call `env::build_worker_env()` instead of inline `.env()` calls
- [ ] Respawn closure in `pool.rs` also uses `env::build_worker_env()` (no duplication)
- [ ] `cargo clippy -p anvilml-worker --features mock-hardware -- -D warnings` exits 0
- [ ] All 8 tests pass with `#[cfg(test)]`
- [ ] No integration tests required (pure functions, no I/O)
- [ ] Backward compatible: existing Python worker still receives `ANVILML_DEVICE_TYPE` and `ANVILML_DEVICE_INDEX` from pool.rs inline calls

### Files to modify:
| File | Action |
|------|--------|
| `crates/anvilml-worker/src/env.rs` | **Create** — new module |
| `crates/anvilml-worker/src/lib.rs` | Modify — add `pub mod env;` |
| `crates/anvilml-worker/src/pool.rs` | Modify — replace inline `.env()` calls with `env::build_worker_env()` in both spawn and respawn paths |

### Files NOT modified:
- `managed.rs` — no changes needed (no env state tracked there)
- `ipc_bridge.rs` — no changes needed (spawn logic is pool-side)
- Python worker code (`worker_main.py`) — no changes needed (receives env vars from Rust side)

---

## Risks & Mitigations

1. **IPEX env var uncertainty**: ENVIRONMENT.md doesn't specify exact IPEX variables. Mitigation: use standard Intel oneAPI `ONEAPI_DEVICE_SELECTOR` pattern; easy to change in isolation since `build_ipex_env()` is a separate function.

2. **Windows ROCm path discovery**: The pip wheel location varies. Mitigation: Phase 1 uses a well-known default (`C:/Program Files/AMD/HIP/bin`) with a comment noting it may need config-driven override in later phases.

3. **Backward compatibility with Python worker**: The existing Python worker reads `ANVILML_DEVICE_TYPE` and `ANVILML_DEVICE_INDEX` from env. These will remain set via inline `.env()` calls in pool.rs, so no Python-side changes needed.

4. **Test isolation**: Since all functions are pure (no I/O), tests are fully isolated. Each test creates its own mock `GpuDevice` and `ServerConfig`. No need for `serial_test` or env var cleanup between tests.
