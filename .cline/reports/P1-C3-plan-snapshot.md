I'll start by reading the necessary context files to understand the task requirements before planning.


Now let me examine the existing code to understand what's already implemented, particularly `worker_main.py` and the IPC protocol:


Now let me check the IPC protocol document and existing tests:


Let me also check if there's an existing `test_worker_messages.py`:


Let me also check the conftest and the Rust messages to understand the exact enum variants:


Now I have a thorough understanding of the codebase. Let me also check if there's an existing plan directory:


Now I have all the context needed. Let me present my plan.

---

## Plan for P1-C3: Python Worker — Ping/Pong + MemoryReport + Shutdown

### Current State Analysis

**What exists already (from P1-C2):**
- `backend/worker/worker_main.py` — fully implemented with CLI parsing, mock mode, Ready event emission, and an IPC read loop
- The read loop (`read_loop()`) at line 201 already handles:
  - **Ping** → sends `Pong { seq }` via `handle_ping()` (line 196) ✅ *already implemented*
  - **Shutdown** → calls `handle_shutdown()` which does `sys.exit(0)` (line 191) ⚠️ *needs fix: doesn't send Dying event first*
  - **Execute** → placeholder Error response (line 220)
  - Unknown → Error response (line 228)
- `backend/worker/ipc.py` — framing layer (`read_message()`, `write_event()`) ✅
- `backend/worker/tests/test_worker_startup.py` — 14 tests, all passing ✅
- **No** `test_worker_messages.py` yet

**What's missing per the task spec:**
1. **Shutdown handler**: Currently just calls `sys.exit(0)` directly. Per IPC_PROTOCOL.md and STARTUP SEQUENCE diagram, it must first emit a `Dying { worker_id, reason }` event before exiting.
2. **MemoryReport background thread**: A background thread that sends `MemoryReport` events every 10 seconds with correct `worker_id`, VRAM fields as non-negative integers.
3. **Tests**: `backend/worker/tests/test_worker_messages.py` with the specified test cases.

### Implementation Plan

#### STEP 1 — Modify `worker_main.py`

**1a. Fix Shutdown handler to emit Dying event before exit**

Current code (line 191-193):
```python
def handle_shutdown():
    sys.exit(0)
```

Replace with:
```python
def handle_shutdown(worker_id: str):
    write_event({
        "type": "Dying",
        "worker_id": worker_id,
        "reason": "Received Shutdown message",
    })
    sys.exit(0)
```

Update the call site in `read_loop()` (line 216) to pass `worker_id`:
```python
if msg_type == "Shutdown":
    handle_shutdown(args.worker_id)  # need worker_id accessible in read_loop
```

To make `worker_id` accessible, we need to either:
- Pass it as a parameter to `read_loop()`, or
- Store it as a module-level variable, or  
- Make `handle_shutdown` accept the worker_id from the outer scope

**Best approach**: Pass `worker_id` into `read_loop()` as an argument. This keeps things clean and testable.

**1b. Add MemoryReport background thread**

Per IPC_PROTOCOL.md:
```json
{
  "type": "MemoryReport",
  "worker_id": "uuid-string",
  "vram_used_mib": 2184,
  "vram_total_mib": 16384,
  "ram_pinned_mib": 0,
  "loaded_models": []
}
```

Implementation:
- Use `threading.Thread` with `daemon=True` (so it doesn't prevent exit)
- Run every 10 seconds using `time.sleep(10)` in a loop
- In mock mode, use the existing mock VRAM values (`MOCK_VRAM_TOTAL_MIB`, derived used from free)
- Send via `write_event()` on each tick
- Stop when a shutdown signal is received (use a `threading.Event` for clean termination)

The thread starts after the Ready event and runs alongside the read loop. The main thread enters the read loop, and when Shutdown arrives → Dying event → exit → daemon thread terminates automatically.

**1c. Thread-safe write_event**

Since two threads (main read loop + memory report thread) will call `write_event()`, we need a `threading.Lock` to prevent interleaved writes on stdout.buffer:
```python
_write_lock = threading.Lock()

def write_event_safe(event: dict) -> None:
    with _write_lock:
        write_event(event)
```

#### STEP 2 — Create `test_worker_messages.py`

Following the same pattern as `test_worker_startup.py` (subprocess-based, frame parsing via `_parse_frames()`), create these tests:

**Test class: `TestPingPongMessage`**
- `test_ping_returns_pong_with_same_seq`: Send Ping with seq=42, verify Pong has seq=42
- `test_ping_multiple_seqs`: Send multiple Ping messages, each gets matching Pong

**Test class: `TestShutdownBehavior`**
- `test_shutdown_emits_dying_event`: Send Shutdown, verify stdout contains Dying event with correct worker_id and reason="Received Shutdown message" before process exits
- `test_shutdown_exits_with_code_0`: Verify returncode is 0 after Shutdown

**Test class: `TestMemoryReport`**
- `test_memory_report_has_correct_worker_id`: Worker emits MemoryReport events containing the correct worker_id
- `test_memory_report_vram_fields_non_negative`: vram_used_mib and vram_total_mib are non-negative integers

For MemoryReport tests, we need to capture stdout over a period longer than 10 seconds. Strategy:
- Spawn worker with stdin closed (no commands), let it run for ~12 seconds
- Parse all frames — first is Ready, subsequent ones should include MemoryReport events
- Verify MemoryReport fields

**Test class: `TestMessageHandling`** (bonus)
- `test_unknown_command_returns_error`: Send a message with unknown type, verify Error event

#### File changes summary

| File | Action | Description |
|------|--------|-------------|
| `backend/worker/worker_main.py` | Modify | Fix Shutdown handler to emit Dying; add MemoryReport daemon thread; add lock for thread safety; pass worker_id through read_loop |
| `backend/worker/tests/test_worker_messages.py` | Create | 6-8 subprocess-based tests |

#### Trade-offs considered

1. **Daemon thread vs. asyncio task**: Since the worker currently uses synchronous blocking I/O (no asyncio), a `threading.Thread` is the simplest approach. If we were to migrate to async, an `asyncio.Task` would be better — but that's out of scope for P1-C3.

2. **Lock granularity**: We only need to protect the `write_event()` call since it does multiple buffer writes (length prefix + body + flush). The lock ensures these happen atomically.

3. **MemoryReport in mock mode**: Uses mock VRAM values (`MOCK_VRAM_TOTAL_MIB=16384`, derived used = total - free ≈ 184 MiB). This is consistent with existing patterns.

4. **Test timing for MemoryReport**: Using a 12-second subprocess timeout ensures at least one MemoryReport fires after the first Ready event. This is reliable since `time.sleep(10)` is deterministic enough for test purposes.

---

**This completes STEP 1 — PLAN.** I am ready to proceed to implementation once approved, which involves:
1. Editing `worker_main.py` (Shutdown fix + MemoryReport thread)
2. Creating `test_worker_messages.py`
3. Running tests to verify all pass with `ANVILML_WORKER_MOCK=1 pytest`