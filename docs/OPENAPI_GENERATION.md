# OPENAPI_GENERATION.md — OpenAPI Type Sharing Pipeline
# AnvilML is the source of truth for all API types.
# BloomeryUI consumes generated TypeScript types. Never hand-write API types.

---

## PIPELINE OVERVIEW

```
anvilml-core types (Rust structs)
         ↓
anvilml-openapi crate generates
         ↓
backend/openapi.json    ← committed to AnvilML repo
         ↓
pnpm generate-types     ← BloomeryUI reads ../backend/openapi.json
         ↓
frontend/src/generated/api-types.ts   ← committed to BloomeryUI repo
         ↓
frontend/src/api/endpoints/*.ts       ← imports from generated types
```

---

## WHEN TO RE-RUN GENERATION

Re-run the full pipeline whenever ANY of these change:
- Any struct in `anvilml-core/src/types/`
- Any handler response type in `anvilml-server`
- Any new endpoint added to the router
- Any WebSocket event type

Cline must run the pipeline as part of any task that modifies API types.
Failure to re-run = TypeScript type errors in frontend = blocked CI.

---

## RUNNING THE PIPELINE

```bash
# Step 1: Regenerate openapi.json (run from backend/)
cargo run -p anvilml-openapi
# Output: backend/openapi.json

# Step 2: Regenerate TypeScript types (run from frontend/)
pnpm generate-types
# Reads:  ../backend/openapi.json
# Output: src/generated/api-types.ts

# Step 3: Verify no TypeScript errors
pnpm type-check
```

---

## anvilml-openapi CRATE

Uses `utoipa` to derive OpenAPI annotations from Rust types and handlers.

```toml
# In anvilml-server/Cargo.toml
[dependencies]
utoipa = { version = "4", features = ["axum_extras", "chrono", "uuid"] }

# In anvilml-openapi/Cargo.toml
[dependencies]
anvilml-server = { path = "../anvilml-server" }
utoipa = "4"
```

### Deriving OpenAPI schemas on types

Every struct in `anvilml-core` that appears in an API response must derive:
```rust
use utoipa::ToSchema;

#[derive(Serialize, Deserialize, ToSchema)]
pub struct JobStatus {
    // ...
}
```

### Annotating handlers

Every handler function must have `#[utoipa::path(...)]` annotation:
```rust
#[utoipa::path(
    get,
    path = "/v1/jobs/{id}",
    responses(
        (status = 200, body = Job),
        (status = 404, body = ApiErrorBody),
    ),
    tag = "jobs"
)]
pub async fn get_job(/* ... */) { /* ... */ }
```

### Generation binary (`anvilml-openapi/src/main.rs`)

```rust
fn main() {
    let spec = ApiDoc::openapi();
    let json = spec.to_pretty_json().expect("failed to serialize openapi spec");
    std::fs::write("openapi.json", json).expect("failed to write openapi.json");
    println!("openapi.json written successfully");
}
```

---

## src/generated/api-types.ts USAGE

```typescript
// Import generated types
import type { components, paths } from '../generated/api-types'

// Use component schemas
type Job = components['schemas']['Job']
type SystemInfo = components['schemas']['SystemInfo']
type WsEvent = components['schemas']['WsEvent']

// Use path-based types for request/response shapes
type SubmitJobRequest = paths['/v1/jobs']['post']['requestBody']['content']['application/json']
type SubmitJobResponse = paths['/v1/jobs']['post']['responses']['202']['content']['application/json']
```

Hand-written convenience aliases live in `src/api/endpoints/*.ts` — they re-export
from generated types with readable names.

```typescript
// src/api/endpoints/jobs.ts
import type { components } from '../../generated/api-types'

export type Job = components['schemas']['Job']
export type JobStatus = components['schemas']['JobStatus']

export async function submitJob(req: SubmitJobRequest): Promise<SubmitJobResponse> {
  return apiFetch('/v1/jobs', { method: 'POST', body: JSON.stringify(req) })
}
```

---

## WEBSOCKET EVENT TYPES

WebSocket events are NOT REST responses, so they need special handling.
They are defined as a discriminated union in `anvilml-core` and exposed
via a special OpenAPI schema named `WsEvent`.

```rust
// In anvilml-core/src/types/events.rs
#[derive(Serialize, Deserialize, ToSchema)]
#[serde(tag = "type")]
pub enum WsEvent {
    #[serde(rename = "job.queued")]
    JobQueued(WsJobQueued),
    #[serde(rename = "worker.ready")]
    WorkerReady(WsWorkerReady),
    // ... all variants
}
```

The generated `WsEvent` TypeScript type is a discriminated union on `type`.

---

## COMMITTED FILES

Both repos commit their generated files:

**AnvilML repo:**
- `backend/openapi.json` — committed, updated when API types change

**BloomeryUI repo:**
- `frontend/src/generated/api-types.ts` — committed, updated after `pnpm generate-types`

CI does NOT regenerate these files. It uses the committed versions.
This means: if a type changes and generation is not re-run and committed,
CI will catch it as a TypeScript error (the generated file will be stale).
