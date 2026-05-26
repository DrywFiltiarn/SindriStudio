# TESTING_STRATEGY.md — SindriStudio Testing Strategy
# Defines mock boundaries, what runs in CI vs locally, and per-layer rules.
# Read this before writing any test code.

---

## CORE PRINCIPLE

Tests must be deterministic and fast. Hardware-dependent tests (GPU, ROCm, CUDA)
cannot run in GitHub Actions CI. The codebase uses a mock boundary at the hardware
and inference layer to ensure the full test suite runs in CI without GPU access.

The mock boundary is a **Rust feature flag** and a **Python environment variable**.

---

## RUST MOCK BOUNDARY

### Feature flag: `mock-hardware`

`anvilml-hardware` has two implementations selected by feature flag:

```toml
# In anvilml-hardware/Cargo.toml
[features]
default = []
mock-hardware = []
```

When `mock-hardware` is active:
- `HardwareDetector::detect()` returns a configurable fake `HardwareInfo`
- Default fake: one CPU device, 8 GiB "VRAM", no GPU
- Can be configured via environment: `ANVILML_MOCK_DEVICE_TYPE=cuda|rocm|ipex|cpu`
- `ANVILML_MOCK_VRAM_MIB=16384` — sets fake VRAM size
- `ANVILML_MOCK_GFX_ARCH=gfx1201` — sets fake GFX arch

All other crates that depend on `anvilml-hardware` automatically get the mock
when `mock-hardware` is enabled.

### CI Cargo config

```yaml
# In CI workflow
- name: Test backend
  run: cargo test --workspace --features mock-hardware
  working-directory: backend
```

### Local (real hardware) testing

```bash
cargo test --workspace    # uses real hardware detection
```

### Test organization per crate

| Crate               | Unit tests          | Integration tests        |
|---------------------|---------------------|--------------------------|
| anvilml-core        | in-file #[cfg(test)]| none needed              |
| anvilml-hardware    | in-file + mock impl | tests/ with mock feature |
| anvilml-registry    | in-file with tempdir| none                     |
| anvilml-ipc         | round-trip encode   | none                     |
| anvilml-worker      | pool logic mocked   | tests/ with mock worker  |
| anvilml-scheduler   | queue logic         | tests/ with mock worker  |
| anvilml-server      | handler unit tests  | tests/ full server + mock worker |
| anvilml-openapi     | schema correctness  | none                     |

---

## PYTHON WORKER MOCK BOUNDARY

### Environment variable: `ANVILML_WORKER_MOCK=1`

When set, the Python worker:
- Does NOT import torch
- Responds to `Execute` messages with a fake `Progress` + `Completed` sequence
- Responds to `Ping` with `Pong`
- Sends a fake `Ready` event immediately on startup
- Sends periodic `MemoryReport` with zeros

This allows the Rust integration tests to spawn a real Python worker process
without needing GPU or PyTorch installed.

```python
# In worker_main.py
import os
MOCK_MODE = os.environ.get("ANVILML_WORKER_MOCK", "0") == "1"

if not MOCK_MODE:
    import torch
    # ... real initialization
```

### Python unit tests (pytest)

Python worker tests run with `ANVILML_WORKER_MOCK=1` by default.
They test: IPC framing, message parsing, executor DAG logic, node base class.
They do NOT test: actual torch inference (that's integration/manual testing).

```bash
# Run Python tests
cd backend
ANVILML_WORKER_MOCK=1 python -m pytest worker/tests/ -v
```

---

## FRONTEND MOCK BOUNDARY

BloomeryUI tests use Vitest with jsdom. No real HTTP requests are made.
All API calls are mocked via `vi.mock()` at the `src/api/client.ts` level.

```typescript
// Standard pattern in component tests
vi.mock('../../api/client', () => ({
  apiFetch: vi.fn(),
}))
```

WebSocket tests mock the `WebSocketManager` singleton:
```typescript
vi.mock('../../api/websocket', () => ({
  wsManager: {
    connect: vi.fn(),
    onEvent: vi.fn(),
    offEvent: vi.fn(),
  }
}))
```

Frontend CI runs `pnpm test:run` — no backend needed.

---

## CI MATRIX

### GitHub Actions jobs

```yaml
jobs:
  backend:
    runs-on: ubuntu-latest
    steps:
      - cargo fmt --check
      - cargo clippy --workspace -- -D warnings --features mock-hardware
      - cargo test --workspace --features mock-hardware
      - cargo run -p anvilml-openapi  # verify openapi.json generates cleanly

  frontend:
    runs-on: ubuntu-latest
    steps:
      - pnpm install --frozen-lockfile
      - pnpm type-check
      - pnpm lint
      - pnpm test:run
      - pnpm build

  python-worker:
    runs-on: ubuntu-latest
    steps:
      - python3.12 -m venv .venv && source .venv/bin/activate
      - pip install msgpack pytest pillow numpy
      # Note: torch NOT installed in CI — mock mode only
      - ANVILML_WORKER_MOCK=1 python -m pytest backend/worker/tests/ -v
```

### Local testing (developer machine, with GPU)

```bash
# Full backend with real hardware
cargo test --workspace

# Python with real torch (after venv setup)
python -m pytest backend/worker/tests/ -v

# Standalone inference test
python backend/scripts/test_inference.py --model-path ../models/diffusion/z-image-turbo
```

---

## TEST NAMING CONVENTIONS

### Rust
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_{unit_of_behavior}() { ... }

    #[tokio::test]
    async fn test_{async_unit_of_behavior}() { ... }
}
```

### Python (pytest)
```python
def test_{unit_of_behavior}():
    ...

class Test{Component}:
    def test_{behavior}(self):
        ...
```

### TypeScript (Vitest)
```typescript
describe('{ComponentOrModule}', () => {
  it('{behavior description}', () => { ... })
})
```

---

## COVERAGE REQUIREMENTS

Phase 1 minimums (not enforced by tooling, enforced by code review / Cline):
- Every public function in every Rust crate has at least one test
- Every API endpoint has at least one happy-path test and one error-path test
- Every IPC message type has a round-trip serialization test
- Every Zustand store action has a unit test
- Every React component that contains logic (not pure display) has a test

Phase 1 explicitly NOT tested (deferred):
- Real GPU inference performance
- Multi-GPU routing
- VRAM eviction under pressure
- Cross-platform IPC on Windows (manual verification initially)

---

## ACCEPTANCE: WHAT "TESTS PASS" MEANS AT COMMIT TIME

For a task to be marked COMPLETE and committed:

1. `cargo test --workspace --features mock-hardware` — exit 0, 0 failures
2. `cargo clippy --workspace -- -D warnings --features mock-hardware` — exit 0
3. `cargo fmt --check` — exit 0
4. `pnpm test:run` — exit 0, 0 failures (if frontend modified)
5. `ANVILML_WORKER_MOCK=1 python -m pytest backend/worker/tests/ -v` — exit 0 (if Python modified)

No exceptions. A single failing test blocks the commit.
