I'm in Plan mode, so I'll explore the codebase, understand the requirements, and create a detailed plan without executing any code changes. Let me start by gathering all necessary context.


Now let me read the task specification and related protocol docs:


Now let me examine the existing anvilml-worker crate to understand what's already in place:


Let me also look at the existing types from anvilml-core and anvilml-ipc that we'll need:


Let me also check the config and server types:


Let me check the existing worker_main.py to understand what's already implemented:


Now let me check the anvilml-hardware mock detector and the existing Cargo.toml for dependencies:


Now let me check the detector trait signature to understand the exact interface:


Let me also check the IPC framing module:


Now I have a comprehensive understanding of the codebase. Let me also verify what Python binary path is available:


Now I have all the context needed. Let me present my plan.

---

## Plan: P1-D1 — WorkerPool + spawn (Plan Only)

### Summary of What We Know

**Task:** Implement `WorkerPool` (pool.rs) and `ManagedWorker` (managed.rs) in `anvilml-worker`. The pool spawns Python worker processes, tracks per-worker state transitions (`Initializing → Idle after Ready event`), and provides acquire/release interfaces. Tests spawn real Python with `ANVILML_WORKER_MOCK=1`.

**Prerequisites already met:** P1-A9 (mock hardware detector) ✅, P1-C2 (Python worker startup + Ready event) ✅

**Existing infrastructure we depend on:**
- `anvilml-core`: `ServerConfig`, `WorkerStatus` (Initializing/Idle/Busy/Unresponsive/Dead), `WorkerInfo`, `GpuDevice`, `HardwareInfo`, `DeviceType`
- `anvilml-ipc`: `WorkerMessage` enum, `WorkerEvent` enum, `write_message()` / `read_message()` framing functions
- `anvilml-hardware`: `MockHardwareDetector` (feature-gated), `detect_all_devices()`, `DeviceDetector` trait
- Python worker: `worker_main.py` — accepts `--worker-id <uuid> --device-index <int>`, emits Ready event, handles Ping/Pong/Shutdown

---

### Design Decisions & Tradeoffs

**1. Process spawn strategy:** Use `tokio::process::Command` to spawn the Python process with stdin/stdout pipes. This gives us async file descriptors for the IPC bridge (which is P1-D2's job, but we need the handles in pool.rs for spawn).

**2. Per-worker state machine:** A simple enum inside `ManagedWorker`:
```
Initializing → Idle ←→ Busy → Dead
                    ↖ Unresponsive (timeout)
```
State transitions are driven by events from the stdout reader and process exit.

**3. Event notification:** Use `tokio::sync::broadcast` for WorkerEvent fan-out. The pool holds a sender; callers call `subscribe_events()` to get receivers. This is lightweight — broadcast with small subscriber count (the scheduler + server).

**4. Worker tracking:** A `HashMap<Uuid, ManagedWorker>` protected by `Arc<Mutex<...>>`. Since the pool's API is async and uses `await` at each call site, a single `Mutex` is fine (no concurrent access to the map from multiple threads within the same pool instance).

**5. Test design — real Python subprocess:** The task spec says "tests spawn real Python with ANVILML_WORKER_MOCK=1." We'll use `tokio::process::Command` in tests, setting the env var, and read from stdout to verify Ready events arrive. If Python is unavailable, we skip with `#[ignore]`.

**6. Cargo.toml dependencies:** The current Cargo.toml already has `anvilml-core`, `anvilml-ipc`, `anvilml-hardware`, `anyhow`. We need to add:
- `tokio` (full) — for async process spawning and IO
- `uuid` (with v4 feature) — for worker ID generation
- `tracing` — for logging
- `serde_json` — for JSON values in messages

---

### Implementation Plan

#### STEP 1: Update `anvilml-worker/Cargo.toml`
Add dependencies: `tokio`, `uuid`, `tracing`, `serde_json`. These are already available as workspace deps.

#### STEP 2: Create `managed.rs` — `ManagedWorker` struct
```rust
pub enum WorkerState {
    Initializing,
    Idle,
    Busy { job_id: Uuid },
    Unresponsive,
    Dead,
}

pub struct ManagedWorker {
    pub id: Uuid,
    pub state: WorkerState,
    pub device_index: u32,
    pub device_type: DeviceType,
    pub pid: Option<u32>,
    // For P1-D2 IPC bridge wiring (handles stored, tasks not yet started):
    stdin_write: Option<tokio::io::Stdin>,
    stdout_read: Option<tokio::io::Stdout>,
}
```

Key methods:
- `new(id, device_index, device_type) -> Self` — creates in Initializing state
- `on_ready(event: &WorkerEvent::Ready)` — transitions to Idle, stores pid info
- `on_event(event: &WorkerEvent)` — handles Pong (no state change), Dying → Dead
- `set_busy(job_id)` / `set_idle()` — explicit status toggles
- `mark_dead(reason)` — transitions to Dead

#### STEP 3: Create `pool.rs` — `WorkerPool` struct
```rust
pub struct WorkerPool {
    config: Arc<ServerConfig>,
    hardware: Arc<HardwareInfo>,
    workers: Arc<Mutex<HashMap<Uuid, ManagedWorker>>>,
    event_tx: broadcast::Sender<WorkerEvent>,
    python_bin: PathBuf,
    worker_script: PathBuf,
}
```

Key methods (matching the task spec API):
- `new(config, hardware) -> Self` — initialize pool, no workers yet
- `spawn_for_device(device) -> Result<Uuid>` — spawn a Python process with correct args/env, return worker UUID
  - Generate UUID for worker_id
  - Build env: set `ANVILML_WORKER_MOCK=1`, `ANVILML_DEVICE_TYPE`, `ANVILML_DEVICE_INDEX`
  - Command: `{python_bin} {worker_script} --worker-id {uuid} --device-index {idx}`
  - Start stdout reader task (one-shot for Ready event in Phase 1)
  - Store worker in HashMap as Initializing
- `spawn_initial_workers()` — spawn `max_workers_per_device` workers per device
- `acquire_idle() -> Option<Uuid>` — find first Idle worker, return its ID
- `set_busy(id, job_id)` / `set_idle(id)` — toggle status
- `send_to(id, msg) -> Result<()>` — write message to worker's stdin (placeholder for P1-D2)
- `subscribe_events() -> broadcast::Receiver<WorkerEvent>` — return subscriber
- `status_snapshot() -> Vec<WorkerInfo>` — convert internal state to API types
- `shutdown_all()` — send Shutdown to all workers, wait briefly

#### STEP 4: Update `lib.rs`
```rust
pub mod pool;
pub mod managed;
```

#### STEP 5: Write tests in `pool.rs` (behind `#[cfg(test)]`)
Tests use mock hardware + real Python subprocess with `ANVILML_WORKER_MOCK=1`:

1. **`test_spawn_creates_entry`** — Spawn a worker, verify it appears in the workers map as Initializing, then becomes Idle after Ready event.
2. **`test_ready_transitions_idle`** — Verify state transitions from Initializing → Idle upon receiving Ready event.
3. **`test_acquire_idle_returns_none_when_empty`** — Pool with no workers returns None.
4. **`test_acquire_idle_returns_worker_id`** — After spawning, acquire_idle returns the UUID of an Idle worker.
5. **`test_acquire_idle_returns_none_when_all_busy`** — Set all workers busy, verify None returned.
6. **`test_set_busy_set_idle_toggle`** — Verify status transitions work correctly.

Test pattern:
```rust
#[tokio::test]
async fn test_spawn_creates_entry() {
    // Use mock hardware detector to get a fake device
    let detector = MockHardwareDetector;
    let hw = detector.detect().await.unwrap().unwrap();
    
    // Create pool with default config
    let config = Arc::new(ServerConfig::defaults());
    let pool = WorkerPool::new(config, Arc::new(hw)).await.unwrap();
    
    // Spawn a worker for the device
    let worker_id = pool.spawn_for_device(&hw.devices[0]).await.unwrap();
    
    // Verify worker exists and is Initializing initially
    {
        let workers = pool.workers.lock().unwrap();
        let worker = workers.get(&worker_id).unwrap();
        assert!(matches!(worker.state, WorkerState::Initializing));
    }
    
    // Wait for Ready event (timeout after 5s)
    let mut rx = pool.subscribe_events();
    tokio::select! {
        event = rx.recv() => {
            if let Ok(WorkerEvent::Ready { worker_id: rid, .. }) = event {
                assert_eq!(rid, worker_id);
            }
        },
        _ = tokio::time::sleep(Duration::from_secs(5)) => {
            panic!("Ready event timeout");
        }
    }
    
    // Now verify worker is Idle
    {
        let workers = pool.workers.lock().unwrap();
        let worker = workers.get(&worker_id).unwrap();
        assert!(matches!(worker.state, WorkerState::Idle));
    }
}
```

#### STEP 6: Handle Python availability gracefully
In tests, check `which python3` exists. If not, mark with `#[ignore]` and log a note.

---

### File List

| Action | File |
|--------|------|
| Modify | `crates/anvilml-worker/Cargo.toml` — add deps |
| Create | `crates/anvilml-worker/src/managed.rs` — ManagedWorker + WorkerState |
| Create | `crates/anvilml-worker/src/pool.rs` — WorkerPool + tests |
| Modify | `crates/anvilml-worker/src/lib.rs` — export modules |

---

### Risks & Mitigations

- **Python not in PATH during test:** Tests use `#[ignore]` guard. CI always has Python 3.12.
- **Stdout reader task lifecycle:** In Phase 1 (this task), we only need to capture the Ready event. The full IPC bridge (P1-D2) will wire up persistent stdin/stdout tasks. For now, spawn a one-shot task that reads until Ready, then hands off or terminates.
- **Process cleanup on test exit:** Ensure spawned Python processes are killed on test teardown (use `tokio::task::JoinHandle` and abort on drop).

---

This completes the plan for P1-D1. The implementation focuses on process spawning, state tracking, and the acquire/release interface — without the full IPC bridge (that's P1-D2). All tests will use real Python subprocesses with `ANVILML_WORKER_MOCK=1` as specified.