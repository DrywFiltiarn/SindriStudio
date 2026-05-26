# API_CONTRACT.md — AnvilML REST + WebSocket API
# Source of truth for all API shapes. Changes here require:
#   1. Update anvilml-core types
#   2. Re-run: cargo run -p anvilml-openapi
#   3. Re-run: pnpm generate-types (in frontend/)
#   4. Verify: pnpm type-check passes

---

## CONVENTIONS

Base URL:     `http://{host}:{port}` — default `http://127.0.0.1:8188`
API prefix:   `/v1/`
Content-Type: `application/json` for all request/response bodies
Dates:        ISO 8601 UTC — `"2025-01-15T14:30:00.000Z"`
IDs:          UUID v4 strings
Phase markers: [P1] implemented Phase 1, [P2] Phase 2

---

## ERROR ENVELOPE (all non-2xx responses)

```json
{ "error": { "code": "JOB_NOT_FOUND", "message": "Job ... does not exist" } }
```

| Code | HTTP |
|---|---|
| `NOT_FOUND` | 404 |
| `BAD_REQUEST` | 400 |
| `INTERNAL` | 500 |
| `JOB_NOT_FOUND` | 404 |
| `MODEL_NOT_FOUND` | 404 |
| `QUEUE_FULL` | 429 |

---

## ENDPOINTS

### GET /health [P1]
```json
{ "status": "ok", "version": "0.1.0", "uptime_seconds": 142 }
```

---

### GET /v1/system [P1]
```json
{
  "host": {
    "system_ram_total_mib": 65536,
    "system_ram_free_mib": 48210,
    "cpu_model": "Intel(R) Xeon(R) CPU E5-2680 v4 @ 2.40GHz",
    "cpu_physical_cores": 14,
    "cpu_logical_threads": 28,
    "cpu_usage_percent": 12.4
  },
  "devices": [
    {
      "index": 0,
      "name": "AMD Radeon RX 9070 XT",
      "device_type": "rocm",
      "gfx_arch": "gfx1201",
      "vram_total_mib": 16384,
      "vram_free_mib": 14200,
      "vram_used_mib": 2184,
      "rebar_enabled": true,
      "driver_version": "7.0.0",
      "capabilities": {
        "fp16": true, "bf16": true, "fp8": true,
        "flash_attention": true, "hipblaslt": true
      }
    }
  ],
  "workers": [
    {
      "id": "uuid",
      "device_index": 0,
      "device_type": "rocm",
      "status": "idle",
      "busy_with_job": null
    }
  ],
  "queue_depth": 0,
  "anvilml_version": "0.1.0"
}
```

`device_type`: `"rocm" | "cuda" | "ipex" | "cpu"`
`workers[].status`: `"initializing" | "idle" | "busy" | "unresponsive" | "dead"`

---

### GET /v1/models [P1]
Query: `?kind=clip|diffusion|vae|lora|controlnet|unet|upscale` (optional)
```json
{
  "models": [{
    "id": "a1b2c3d4e5f6",
    "filename": "z-image-turbo",
    "path": "diffusion/z-image-turbo",
    "kind": "diffusion",
    "size_bytes": 3800000000,
    "size_mib": 3624,
    "dtype_hint": "bfloat16",
    "sha256": null
  }],
  "total": 1
}
```

### GET /v1/models/:id [P1]
Single model. 404 `MODEL_NOT_FOUND` if not found.

---

### POST /v1/jobs [P1]
Request:
```json
{
  "workflow": { "nodes": [], "edges": [] },
  "settings": {
    "dtype": "bfloat16",
    "use_flash_attention": true,
    "vram_budget_mib": 0,
    "allow_cpu_offload": false
  }
}
```
Response 202:
```json
{
  "job_id": "uuid",
  "status": "queued",
  "queued_at": "2025-01-15T14:30:00.000Z",
  "position_in_queue": 0
}
```

### GET /v1/jobs [P1]
Query: `?status=queued|running|completed|failed|cancelled` `?limit=50`
```json
{
  "jobs": [{
    "id": "uuid",
    "status": "completed",
    "queued_at": "...", "started_at": "...", "completed_at": "...",
    "worker_id": "uuid",
    "error": null,
    "artifact_count": 1
  }],
  "total": 1
}
```

`status` values: `"queued" | "prefetching" | "dispatched" | "running" | "completed" | "failed" | "cancelled"`

### GET /v1/jobs/:id [P1]
Single job. 404 if not found.

### DELETE /v1/jobs/:id [P1]
```json
{ "cancelled": true, "job_id": "uuid" }
```
400 if already completed or cancelled.

---

### GET /v1/workers [P1]
```json
{
  "workers": [{
    "id": "uuid",
    "device_index": 0,
    "device_type": "rocm",
    "status": "idle",
    "busy_with_job": null,
    "pid": 12345,
    "vram_used_mib": 0,
    "ram_pinned_mib": 0,
    "loaded_models": []
  }]
}
```

---

### GET /v1/artifacts [P2]
Query: `?job_id=uuid` (optional)
```json
{
  "artifacts": [{
    "hash": "abc123def456...",
    "job_id": "uuid",
    "url_path": "/v1/artifacts/abc123...",
    "width": 1024, "height": 1024,
    "size_bytes": 2048000,
    "created_at": "...",
    "prompt": "a red fox",
    "seed": 42,
    "steps": 8
  }],
  "total": 1
}
```

### GET /v1/artifacts/:hash [P2]
Serves PNG file directly. Content-Type: image/png
Use as `<img src="http://server/v1/artifacts/{hash}">`.

---

## WEBSOCKET — GET /v1/events [P1]

`ws://127.0.0.1:8188/v1/events`
All messages: JSON text frames. Discriminator field: `"type"`.
Server sends `system.stats` every 5 seconds automatically.

### job.queued [P1]
```json
{ "type": "job.queued", "ts": "...", "job_id": "uuid", "position": 0 }
```

### job.started [P1]
```json
{ "type": "job.started", "ts": "...", "job_id": "uuid", "worker_id": "uuid" }
```

### job.progress [P2]
```json
{
  "type": "job.progress", "ts": "...", "job_id": "uuid",
  "node_id": "n3", "percent": 45.0, "step": 9, "total_steps": 20
}
```

### job.image_ready [P2]
```json
{
  "type": "job.image_ready", "ts": "...", "job_id": "uuid",
  "artifact_hash": "abc123...",
  "image_url": "/v1/artifacts/abc123...",
  "width": 1024, "height": 1024,
  "prompt": "a red fox", "seed": 42, "steps": 8
}
```

### job.completed [P1]
```json
{
  "type": "job.completed", "ts": "...", "job_id": "uuid",
  "artifact_count": 1
}
```

### job.failed [P1]
```json
{
  "type": "job.failed", "ts": "...", "job_id": "uuid",
  "error": "OOM", "traceback": "..."
}
```

### job.cancelled [P1]
```json
{ "type": "job.cancelled", "ts": "...", "job_id": "uuid" }
```

### worker.ready [P1]
```json
{
  "type": "worker.ready", "ts": "...", "worker_id": "uuid",
  "device_index": 0, "device_type": "rocm",
  "torch_version": "2.7.0+rocm7.0", "gfx_arch": "gfx1201",
  "num_threads": 14, "num_interop_threads": 4
}
```

### worker.memory [P1]
```json
{
  "type": "worker.memory", "ts": "...", "worker_id": "uuid",
  "vram_used_mib": 2184, "vram_total_mib": 16384,
  "ram_pinned_mib": 0, "loaded_models": []
}
```

### system.stats [P1]
```json
{
  "type": "system.stats", "ts": "...",
  "cpu_usage_percent": 12.4, "ram_free_mib": 48210,
  "gpu_stats": [{ "index": 0, "vram_used_mib": 2184, "vram_total_mib": 16384 }]
}
```
