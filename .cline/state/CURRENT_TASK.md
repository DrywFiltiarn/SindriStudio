# Current Task

**Task:** P1-D2 — IPC bridge (stdin writer + stdout reader)
**Status:** ✅ COMPLETE — all tests pass, zero failures
**Step:** 4-DONE ✅

## Completed
- [x] Created `ipc_bridge.rs` with `spawn_stdin_writer()` and `spawn_stdout_reader()`
- [x] Modified `managed.rs` — added `msg_tx`, `stdin_reader_handle`, `stdout_reader_handle` fields
- [x] Modified `pool.rs` — extracted inline stdout reader into bridge, wired up stdin writer, replaced stub `send_to()`, added respawn logic with 2s delay, updated `shutdown_all()`
- [x] Modified `lib.rs` — added `pub mod ipc_bridge;`
- [x] All tests pass: 6 passed, 0 failed, 3 ignored (require Python)
- [x] Workspace compiles cleanly (anvilml-server depends on anvilml-worker)

## Files Changed
| File | Action |
|------|--------|
| `crates/anvilml-worker/src/ipc_bridge.rs` | **Created** — stdin writer + stdout reader tasks |
| `crates/anvilml-worker/src/managed.rs` | **Modified** — added msg_tx, handle fields |
| `crates/anvilml-worker/src/pool.rs` | **Modified** — bridge integration, send_to, respawn |
| `crates/anvilml-worker/src/lib.rs` | **Modified** — pub mod ipc_bridge |

## Test Results
```
cargo test -p anvilml-worker
test result: ok. 6 passed; 0 failed; 3 ignored
```
