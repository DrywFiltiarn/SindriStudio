# IPC_PROTOCOL.md — AnvilML Rust ↔ Python Worker IPC Protocol
# This is a BINARY CONTRACT. Field names are exact. Types are exact.
# Any deviation causes silent deserialization failures.
# Changes here require matching changes in BOTH:
#   backend/crates/anvilml-ipc/src/messages.rs
#   backend/worker/ipc.py

---

## TRANSPORT

| Property       | Value                                                     |
|----------------|-----------------------------------------------------------|
| Rust→Python    | Written to worker's **stdin**                             |
| Python→Rust    | Written to worker's **stdout**                            |
| Framing        | 4-byte big-endian u32 length prefix + msgpack body        |
| Python streams | `sys.stdin.buffer` and `sys.stdout.buffer` (binary)       |
| Flush          | Python MUST call `sys.stdout.buffer.flush()` after every write |
| Max msg size   | 64 MiB (enforced by Rust reader — larger = error)         |

---

## FRAMING

### Rust (anvilml-ipc/src/framing.rs)
```rust
pub async fn write_message<W: AsyncWrite + Unpin, T: Serialize>(
    writer: &mut W, msg: &T
) -> Result<()> {
    let bytes = rmp_serde::to_vec_named(msg)?;
    let len = (bytes.len() as u32).to_be_bytes();
    writer.write_all(&len).await?;
    writer.write_all(&bytes).await?;
    Ok(())
}

pub async fn read_message<R: AsyncRead + Unpin, T: DeserializeOwned>(
    reader: &mut R
) -> Result<T> {
    let mut len_buf = [0u8; 4];
    reader.read_exact(&mut len_buf).await?;
    let len = u32::from_be_bytes(len_buf) as usize;
    ensure!(len <= 64 * 1024 * 1024, "message too large: {} bytes", len);
    let mut buf = vec![0u8; len];
    reader.read_exact(&mut buf).await?;
    Ok(rmp_serde::from_slice(&buf)?)
}
```

### Python (worker/ipc.py)
```python
import sys, struct, msgpack

def read_message() -> dict:
    raw = sys.stdin.buffer.read(4)
    if len(raw) < 4:
        raise EOFError("supervisor closed stdin")
    length = struct.unpack(">I", raw)[0]
    body = sys.stdin.buffer.read(length)
    return msgpack.unpackb(body, raw=False)

def write_event(event: dict) -> None:
    body = msgpack.packb(event, use_bin_type=True)
    sys.stdout.buffer.write(struct.pack(">I", len(body)))
    sys.stdout.buffer.write(body)
    sys.stdout.buffer.flush()  # NEVER omit
```

---

## SERIALIZATION RULES

- Discriminator key: `"type"` (string) in BOTH directions
- Rust serde config: `#[serde(tag = "type")]` on enums
- UUID fields: serialized as strings — `"550e8400-e29b-41d4-a716-446655440000"`
- Optional fields: always present in message, value is msgpack nil (`None` in Python)
  Never omit an optional key — always send the key with nil value
- Integer types: msgpack integers (not strings)
- Float types: msgpack float64
- Boolean: msgpack bool
- Timestamps: NOT used in IPC — only in REST API responses

---

## RUST → PYTHON: WorkerMessage

### Execute
```json
{
  "type": "Execute",
  "job_id": "uuid-string",
  "graph": { "nodes": [], "edges": [] },
  "settings": {
    "dtype": "bfloat16",
    "use_flash_attention": true,
    "vram_budget_mib": 14000,
    "allow_cpu_offload": false
  }
}
```
| Field | Type | Values |
|---|---|---|
| `type` | string | `"Execute"` |
| `job_id` | string (UUID) | |
| `graph` | map | arbitrary workflow |
| `settings.dtype` | string | `"float32"` \| `"float16"` \| `"bfloat16"` |
| `settings.use_flash_attention` | bool | |
| `settings.vram_budget_mib` | u64 | 0 = no limit |
| `settings.allow_cpu_offload` | bool | |

### Cancel
```json
{ "type": "Cancel", "job_id": "uuid-string" }
```

### PreloadModel
```json
{
  "type": "PreloadModel",
  "model_path": "diffusion/z-image-turbo",
  "dtype": "bfloat16"
}
```

### EvictModel
```json
{ "type": "EvictModel", "model_path": "diffusion/z-image-turbo" }
```

### Ping
```json
{ "type": "Ping", "seq": 1 }
```
| Field | Type |
|---|---|
| `seq` | u64 |

### Shutdown
```json
{ "type": "Shutdown" }
```

---

## PYTHON → RUST: WorkerEvent

### Ready
Sent once after startup is complete. Rust MUST NOT dispatch jobs until Ready received.
```json
{
  "type": "Ready",
  "worker_id": "uuid-string",
  "device_index": 0,
  "device_type": "rocm",
  "torch_version": "2.7.0+rocm7.0",
  "rocm_version": "7.0.0",
  "cuda_version": null,
  "gfx_arch": "gfx1201",
  "device_name": "AMD Radeon RX 9070 XT",
  "num_threads": 14,
  "num_interop_threads": 4,
  "vram_total_mib": 16384,
  "vram_free_mib": 16200,
  "mock_mode": false
}
```
| Field | Type | Notes |
|---|---|---|
| `type` | string | `"Ready"` |
| `worker_id` | string (UUID) | must match `--worker-id` arg |
| `device_index` | u32 | must match `--device-index` arg |
| `device_type` | string | `"rocm"` \| `"cuda"` \| `"ipex"` \| `"cpu"` |
| `torch_version` | string | `torch.__version__` |
| `rocm_version` | string \| nil | `torch.version.hip` or nil |
| `cuda_version` | string \| nil | `torch.version.cuda` or nil |
| `gfx_arch` | string \| nil | from device properties or nil |
| `device_name` | string \| nil | GPU name or nil |
| `num_threads` | u32 | `torch.get_num_threads()` after setting |
| `num_interop_threads` | u32 | `torch.get_num_interop_threads()` |
| `vram_total_mib` | u64 | 0 if CPU |
| `vram_free_mib` | u64 | 0 if CPU |
| `mock_mode` | bool | true if `ANVILML_WORKER_MOCK=1` |

### Progress
```json
{
  "type": "Progress",
  "job_id": "uuid-string",
  "node_id": "n3",
  "percent": 45.0,
  "step": 9,
  "total_steps": 20
}
```

### ImageReady
```json
{
  "type": "ImageReady",
  "job_id": "uuid-string",
  "image_b64": "<base64 PNG>",
  "width": 1024,
  "height": 1024,
  "format": "png",
  "seed": 42,
  "steps": 8,
  "prompt": "a red fox in a snowy forest"
}
```
| Field | Type | Notes |
|---|---|---|
| `image_b64` | string | base64-encoded PNG bytes |
| `format` | string | `"png"` always in Phase 2 |
| `seed` | i64 | actual seed used; -1 if unknown |

### Completed
```json
{
  "type": "Completed",
  "job_id": "uuid-string",
  "artifact_paths": []
}
```
Note: artifact_paths is always empty — artifacts are delivered via ImageReady.
Kept for protocol completeness and future use.

### Failed
```json
{
  "type": "Failed",
  "job_id": "uuid-string",
  "error": "short description",
  "traceback": "full Python traceback string"
}
```

### ModelPreloaded
```json
{
  "type": "ModelPreloaded",
  "model_path": "diffusion/z-image-turbo",
  "vram_used_mib": 0,
  "ram_used_mib": 22697
}
```

### MemoryReport
Sent automatically every 10 seconds and after any model load/evict.
```json
{
  "type": "MemoryReport",
  "worker_id": "uuid-string",
  "vram_used_mib": 2184,
  "vram_total_mib": 16384,
  "ram_pinned_mib": 0,
  "loaded_models": [
    {
      "path": "diffusion/z-image-turbo",
      "dtype": "bfloat16",
      "vram_mib": 2184,
      "ram_mib": 0,
      "last_used_secs_ago": 5
    }
  ]
}
```

### Pong
```json
{ "type": "Pong", "seq": 1 }
```

### Dying
```json
{
  "type": "Dying",
  "worker_id": "uuid-string",
  "reason": "Received Shutdown message"
}
```

---

## STARTUP SEQUENCE

```
Rust supervisor                     Python worker
     |                                   |
     |── spawn (stdin/stdout pipes) ────>|
     |                                   |── set OMP/MKL env vars
     |                                   |── import torch (or mock)
     |                                   |── configure device
     |                                   |── query VRAM
     |<──────── Ready event ─────────────|
     |                                   |── enter IPC read loop
     |── (jobs may now be dispatched) ──>|
     |                                   |
     |<──── MemoryReport (every 10s) ────|
     |                                   |
     |── Execute { job_id, graph } ─────>|
     |<──── Progress events ─────────────|
     |<──── ImageReady event ────────────|
     |<──── Completed event ─────────────|
     |                                   |
     |── Shutdown ────────────────────── >|
     |<──── Dying event ─────────────────|
     |                          (exit 0) |
```

---

## ERROR HANDLING

| Situation | Worker action |
|---|---|
| Malformed msgpack received | Log to stderr, send `Failed` if job active, continue loop |
| torch device unavailable | Send `Dying { reason: "Device unavailable" }`, exit 1 |
| OOM during inference | Catch, send `Failed` with traceback, continue loop |
| Unhandled exception in node | Catch, send `Failed` with traceback, continue loop |
| stdin closed | Exit cleanly (supervisor terminated) |
| SIGTERM / SIGINT | Send `Dying`, exit 0 |
