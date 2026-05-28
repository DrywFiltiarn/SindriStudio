# P1-E1 Plan — JobQueue + VramLedger

**Task:** anvilml-scheduler: JobQueue + VramLedger  
**Phase:** 1  
**Step:** 1-PLAN  
**Status:** COMPLETE  

---

## Context

- Prerequisites: P1-A4 (core types) — confirmed complete
- Current crate state: skeleton `lib.rs`, empty `Cargo.toml` with deps
- Files to create: `queue.rs`, `ledger.rs`
- Pure logic, no async/IO
- Acceptance: `cargo test -p anvilml-scheduler` exits 0 with ≥8 tests

## Design Decisions

1. **JobQueue**: `VecDeque<Job>` with FIFO ordering (per spec). O(n) cancel/get via linear scan — acceptable for Phase 1.
2. **VramLedger**: HashMap-based per-device tracking exactly as specified in TASKS_PHASE1.md.
3. **Tests**: 12 total — 6 for JobQueue, 6 for VramLedger (exceeds ≥8 minimum).
4. **Dependencies**: Keep existing Cargo.toml deps; only `anvilml-core` is needed for this task.

## Implementation Steps

- **STEP 1** (this step): Write plan report ✅
- **STEP 2**: Implement `queue.rs` — JobQueue struct with push/pop_front/cancel/get/list/len + 6 tests
- **STEP 3**: Implement `ledger.rs` — VramLedger struct with used_mib/free_mib/would_fit/mark_loaded/mark_evicted + 6 tests
- **STEP 4**: Update `lib.rs` module declarations, verify with `cargo test -p anvilml-scheduler`

## Files to Create/Modify

| File | Action |
|------|--------|
| `crates/anvilml-scheduler/src/queue.rs` | **Create** — JobQueue + 6 tests |
| `crates/anvilml-scheduler/src/ledger.rs` | **Create** — VramLedger + 6 tests |
| `crates/anvilml-scheduler/src/lib.rs` | **Modify** — add module declarations |
