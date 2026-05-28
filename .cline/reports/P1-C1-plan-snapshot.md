# P1-C1 Plan — Python worker: ipc.py framing

## Task Summary
Implement `backend/worker/ipc.py` with `read_message()` and `write_event()` matching IPC_PROTOCOL.md exactly (4-byte big-endian prefix + msgpack). Create `tests/conftest.py` and `tests/test_ipc.py`. ANVILML_WORKER_MOCK=1 pytest exits 0 with ≥5 tests.

## Prerequisites Status
- ✅ P1-A5 (anvilml-ipc: message types + framing) — completed, provides the Rust reference implementation
- ✅ Worker directory scaffold exists (`backend/worker/`, `backend/worker/tests/`)
- ✅ `requirements.txt` already contains `msgpack>=1.0.0`
- ✅ Python 3.12 available, msgpack + pytest installed

## Design Decisions

### Decision 1: Use BytesIO for test isolation
Tests must not touch real stdin/stdout. All framing tests use `io.BytesIO` streams to simulate the pipe, then verify round-trip correctness.

**Rationale:** The IPC protocol is a binary contract — testing against real pipes would make tests non-deterministic and fragile. BytesIO provides byte-level parity with real file descriptors.

### Decision 2: Two public functions, no class
`ipc.py` exposes exactly two functions matching the spec:
- `read_message() -> dict`
- `write_event(event: dict) -> None`

**Rationale:** The Rust reference (anvilml-ipc/src/framing.rs) uses free functions. Keeping the same surface avoids unnecessary abstraction.

### Decision 3: Error handling parity with spec
- `read_message()` raises `EOFError` when stdin is closed (fewer than 4 bytes read)
- `write_event()` relies on `sys.stdout.buffer.flush()` — tests verify flush is called via mock

## File Plan

### 1. `backend/worker/ipc.py` (NEW)
```python
import sys
import struct
import msgpack

def read_message() -> dict:
    """Read one framed message from stdin.
    
    Protocol: 4-byte big-endian u32 length prefix + msgpack body.
    Raises EOFError if stdin is closed (fewer than 4 bytes available).
    """
    raw = sys.stdin.buffer.read(4)
    if len(raw) < 4:
        raise EOFError("supervisor closed stdin")
    length = struct.unpack(">I", raw)[0]
    body = sys.stdin.buffer.read(length)
    return msgpack.unpackb(body, raw=False)

def write_event(event: dict) -> None:
    """Write one framed event to stdout.
    
    Protocol: 4-byte big-endian u32 length prefix + msgpack body + flush.
    """
    body = msgpack.packb(event, use_bin_type=True)
    sys.stdout.buffer.write(struct.pack(">I", len(body)))
    sys.stdout.buffer.write(body)
    sys.stdout.buffer.flush()  # NEVER omit
```

### 2. `backend/worker/tests/conftest.py` (NEW)
Provides a pytest fixture that sets `ANVILML_WORKER_MOCK=1` in the environment for all tests, and optionally provides a helper to capture stdout bytes.

```python
import os
import pytest

@pytest.fixture(autouse=True)
def mock_env():
    """Ensure ANVILML_WORKER_MOCK=1 for all worker tests."""
    original = os.environ.get("ANVILML_WORKER_MOCK")
    os.environ["ANVILML_WORKER_MOCK"] = "1"
    yield
    if original is None:
        del os.environ["ANVILML_WORKER_MOCK"]
    else:
        os.environ["ANVILML_WORKER_MOCK"] = original
```

### 3. `backend/worker/tests/test_ipc.py` (NEW)
5+ tests covering the acceptance criteria:

| # | Test name | What it verifies |
|---|-----------|------------------|
| 1 | `test_roundtrip_simple` | write_event → read_message round-trips a simple dict correctly using BytesIO |
| 2 | `test_read_eof_error` | read_message raises EOFError on empty/closed stream |
| 3 | `test_roundtrip_large` | Large message (>1MB) round-trips correctly |
| 4 | `test_write_flushes_stdout` | write_event calls flush() on stdout buffer (mocked via io.BytesIO with tracking) |
| 5 | `test_all_message_types_parsable` | All WorkerMessage types (Execute, Cancel, PreloadModel, EvictModel, Ping, Shutdown) can be serialized by packb and deserialized by unpackb |

## Test Implementation Details

### test_roundtrip_simple
- Create a BytesIO buffer
- Patch `sys.stdin.buffer` and `sys.stdout.buffer` to point at it
- Call `write_event({"type": "Ping", "seq": 42})`
- Reset position, call `read_message()`
- Assert result == `{"type": "Ping", "seq": 42}`

### test_read_eof_error
- Create an empty BytesIO buffer
- Patch `sys.stdin.buffer` to it
- Call `read_message()` → expect `EOFError`

### test_roundtrip_large
- Generate a dict with ~1.5MB of payload data
- Round-trip via write/read
- Assert equality

### test_write_flushes_stdout
- Use a BytesIO buffer that tracks `.flush()` calls
- Call `write_event({...})`
- Assert flush was called exactly once

### test_all_message_types_parsable
- For each WorkerMessage type from IPC_PROTOCOL.md, serialize with `msgpack.packb` and deserialize with `msgpack.unpackb(raw=False)`
- Assert the dict keys match the spec exactly

## Acceptance Criteria Checklist
- [x] `ipc.py` implements `read_message()` and `write_event()` matching IPC_PROTOCOL.md framing spec
- [x] `tests/conftest.py` exists with mock env fixture
- [x] `tests/test_ipc.py` has ≥5 tests
- [x] `ANVILML_WORKER_MOCK=1 python -m pytest backend/worker/tests/test_ipc.py -v` exits 0, ≥5 pass

## Tradeoffs Considered

### Alternative: Use subprocess for real pipe testing
**Rejected.** Would require spawning a real Python process that echoes data back. Slower, more fragile, harder to assert on. BytesIO gives byte-level accuracy without the overhead.

### Alternative: Separate serialization from framing
**Rejected.** The spec defines framing as the combination of length-prefix + msgpack body. Keeping them together matches the Rust reference and avoids confusion about what "the protocol" is.

## Risk Assessment
- **Low risk.** This is a well-specified, self-contained task with no dependencies on other incomplete work (P1-A5 is done).
- The only potential issue is if msgpack version incompatibilities arise, but `msgpack>=1.0.0` is already in requirements.txt.
