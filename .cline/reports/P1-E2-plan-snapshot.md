I'll start by reading the required context files to understand the task fully.


Now let me examine the existing scheduler crate to understand what's already in place from P1-E1:


Now let me examine the existing types and worker pool to understand the interfaces:


Now let me check the hardware detector trait and mock implementation:


Let me also check the events types and the managed worker to understand state transitions:


Now let me check the registry crate to understand its interface:


Now I have a thorough understanding of the codebase. Let me also check if there's an existing `.cline/reports` directory and the hardware crate Cargo.toml:


I now have a comprehensive understanding of the entire codebase. Let me present my plan.

---

## Plan: P1-E2 — anvilml-scheduler: JobScheduler dispatch loop + DAG

### 1. Context Summary

P1-E1 is complete (JobQueue with 6 tests, VramLedger with 6 tests). The `anvilml-scheduler` crate already has:
- `queue.rs` — `JobQueue` (FIFO `VecDeque<Job>`, push/pop/cancel/get/list/len)
- `ledger.rs` — `VramLedger` (per-device VRAM budget tracking, used/free/would_fit/mark_loaded/mark_evicted)

I need to add two new modules:
- **`dag.rs`** — topological sort + cycle detection for workflow graphs
- **`scheduler.rs`** — `JobScheduler` that wires queue, ledger, worker pool, and model registry together with a dispatch loop and status broadcast

### 2. Files to Create / Modify

#### A. New files:

1. **`crates/anvilml-scheduler/src/dag.rs`** — DAG validation + topological sort
2. **`crates/anvilml-scheduler/src/scheduler.rs`** — JobScheduler struct + dispatch loop
3. **`.cline/reports/P1-E2.md`** — plan report

#### B. Modified files:

4. **`crates/anvilml-scheduler/src/lib.rs`** — add `pub mod dag; pub mod scheduler;`
5. **`crates/anvilml-scheduler/Cargo.toml`** — add `tokio`, `tracing`, `serde_json` dependencies; add `mock-hardware` feature

### 3. Detailed Design

#### 3.1 `dag.rs` — Topological Sort + Cycle Detection

**Input:** A `serde_json::Value` representing a workflow graph:
```json
{ "nodes": [{"id": "n1", ...}, {"id": "n2", ...}], "edges": [{"from": "n1", "to": "n2"}] }
```

**Output:** `Result<Vec<String>>` — list of node IDs in valid execution order, or an error if a cycle is detected.

**Algorithm:** Kahn's algorithm (BFS-based topological sort):
1. Parse nodes into a set of IDs and build an adjacency map from edges
2. Compute in-degree for each node
3. Seed queue with all zero in-degree nodes
4. BFS: pop node, add to result, decrement in-degrees of neighbors
5. If result length < total nodes → cycle detected → return `Err`

**Tests (≥4 tests):**
- `test_topo_sort_linear` — `n1→n2→n3` returns `[n1, n2, n3]`
- `test_topo_sort_diamond` — diamond DAG returns valid order
- `test_cycle_detection_1` — `n1→n2→n1` returns Err
- `test_cycle_detection_self_loop` — `n1→n1` returns Err
- `test_empty_graph` — empty nodes/edges returns empty vec

#### 3.2 `scheduler.rs` — JobScheduler Dispatch Loop

**Core struct:**
```rust
pub struct JobScheduler {
    workers: Arc<WorkerPool>,
    registry: Arc<ModelRegistry>,
    hardware: Arc<HardwareInfo>,
    queue: Mutex<JobQueue>,
    ledger: Mutex<VramLedger>,
    jobs: RwLock<HashMap<Uuid, Job>>,       // all submitted jobs for lookup
    status_tx: broadcast::Sender<JobStatusEvent>,
    handle: Mutex<Option<JoinHandle<()>>>,   // dispatch loop task handle
    stopped: AtomicBool,
}
```

**Public API:**
- `new(workers, registry, hardware, db)` — constructor; initializes ledger budgets from hardware VRAM
- `submit(workflow, settings) → Result<SubmitJobResponse>` — validates DAG, creates Job, queues it, broadcasts `JobQueued` event
- `cancel(job_id) → Result<()>` — sets job status to Cancelled, removes from queue
- `get_job(id) → Option<Job>` — lookup in jobs map
- `list_jobs(status_filter) → Vec<Job>` — filter and return all jobs
- `start_dispatch_loop(self: Arc<Self>)` — spawns a tokio task that loops: find idle worker → check VRAM → assign job → transition worker to busy → update job status
- `stop()` — sets stopped flag, joins handle
- `subscribe_status() → broadcast::Receiver<JobStatusEvent>`

**JobStatusEvent:** A simple event type for the broadcast channel:
```rust
pub struct JobStatusEvent {
    pub job_id: Uuid,
    pub status: JobStatus,
}
```

**Dispatch loop logic (simplified, no real inference):**
1. While not stopped:
   - Pop front job from queue if any
   - Find an idle worker via `workers.acquire_idle()`
   - Check VRAM fits via ledger
   - Update job status → Dispatched/Running, set worker_id, started_at
   - Mark worker busy
   - Broadcast status event
   - Sleep briefly (e.g., 100ms) to avoid tight spinning

**Mock strategy for tests:** The dispatch loop test needs a mock WorkerPool. Since the real `WorkerPool` spawns actual Python processes, we'll use a **simplified in-memory test approach**:
- For DAG tests: pure function, no mocking needed
- For scheduler submit/cancel/list/get: create a `JobQueue` + `VramLedger` directly, test the core logic without needing a full WorkerPool
- For dispatch loop assignment: we'll use the real `WorkerPool` with mock hardware (`#[ignore]` if Python unavailable, or test the queue+ledger logic in isolation)

**More pragmatic approach for ≥6 passing tests:**
Since spawning real workers is expensive and requires Python, I'll design the scheduler to be **testable without full worker integration**:

1. **DAG tests (4 tests)** — pure function, no mocking
2. **Scheduler submit test (1 test)** — create a `JobQueue` directly, push a job, verify it's there
3. **Scheduler cancel test (1 test)** — cancel removes from queue and updates status
4. **Scheduler list/get test (1 test)** — CRUD on jobs map
5. **Scheduler VRAM check test (1 test)** — ledger integration with scheduler
6. **Scheduler dispatch loop assignment test (1 test)** — uses real `WorkerPool` with mock hardware, spawns a mock Python worker, verifies job gets assigned to idle worker

This gives us exactly 7 tests, well above the ≥6 threshold.

#### 3.3 Dependencies to Add to `Cargo.toml`

```toml
[dependencies]
tokio       = { workspace = true, features = ["full"] }
tracing     = { workspace = true }
serde_json  = { workspace = true }

[features]
default = []
mock-hardware = []

[dev-dependencies]
tempfile    = "3"
```

Note: `anvilml-worker` and `anvilml-registry` are already listed as dependencies from P1-E1. We need to add `tokio`, `tracing`, `serde_json`. Also add the `mock-hardware` feature so tests can use mock hardware.

### 4. Tradeoffs Considered

| Decision | Option A (chosen) | Option B | Rationale |
|----------|-------------------|----------|-----------|
| Dispatch loop as tokio task | Spawn via `start_dispatch_loop(self: Arc<Self>)` | Return a struct with `.tick()` method that callers poll | Task-based is simpler for background operation; matches the API spec in TASKS_PHASE1.md |
| Test isolation | DAG = pure functions, scheduler = direct queue/ledger testing | Try to mock WorkerPool trait | No need to over-engineer mocks for Phase 1; real WorkerPool with mock Python worker works fine |
| Status broadcast | `broadcast::Sender<JobStatusEvent>` | Use the existing `WorkerPool` event channel | Separate channel keeps scheduler concerns decoupled from worker events |

### 5. Implementation Steps (for Act mode)

**STEP 1:** Update `Cargo.toml` — add `tokio`, `tracing`, `serde_json`; add `[features]` section with `mock-hardware`

**STEP 2:** Create `dag.rs` — implement `topological_sort(workflow: &Value) -> Result<Vec<String>>` with Kahn's algorithm + 4 tests

**STEP 3:** Update `lib.rs` — add `pub mod dag; pub mod scheduler;`

**STEP 4:** Create `scheduler.rs` — implement `JobScheduler` struct, JobStatusEvent, submit/cancel/get/list/start_dispatch_loop/stop/subscribe_status with 3+ tests

**STEP 5:** Run `cargo test -p anvilml-scheduler --features mock-hardware` — verify ≥6 tests pass and exit 0

**STEP 6:** Update `.cline/state/CURRENT_TASK.md` and write `.cline/reports/P1-E2.md`

---

Does this plan align with your expectations? Any adjustments before I proceed to implementation?