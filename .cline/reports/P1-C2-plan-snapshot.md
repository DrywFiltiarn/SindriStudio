# P1-C2 Plan — Python worker: worker_main.py startup + Ready

**Task ID**: P1-C2
**Phase**: 1
**Prerequisites**: P1-C1 (ipc.py framing) ✅
**Step**: 1-PLAN
**Status**: COMPLETE (plan written, awaiting approval)

## Plan Summary

### Files to create:
1. `backend/worker/worker_main.py` — Main entry point with CLI parsing, startup sequence, Ready event, IPC read loop
2. `backend/worker/tests/test_worker_startup.py` — 5+ tests for startup + Ready event (all mock mode)

### Key design decisions:
- Mock mode (`ANVILML_WORKER_MOCK=1`): skip torch import entirely, use fake defaults
- Real mode: set OMP/MKL env vars BEFORE `import torch`, then detect device and query VRAM
- Device type detection from environment variables injected by Rust supervisor
- Single-file architecture (intentional — executor.py comes later)
- Tests use subprocess to spawn worker_main.py, parse stdout frames via IPC framing

### Test plan (5 tests):
1. Mock mode sends Ready event with mock_mode=true
2. Ready event contains correct worker_id from CLI arg
3. Ready event contains correct device_index from CLI arg
4. Ready event has positive num_threads and num_interop_threads
5. Worker exits cleanly when stdin is closed

### Acceptance:
- `ANVILML_WORKER_MOCK=1 python -m pytest backend/worker/tests/test_worker_startup.py -v` exits 0 with ≥5 tests passing
