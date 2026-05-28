# Task Report: P1-B2 — anvilml-registry: SQLite persistence

## Summary

Implemented full SQLite persistence layer for the AnvilML model registry. The `ModelRegistry` is now async-backed by a `SqlitePool` with proper schema initialization, CRUD operations, and idempotent scan-and-persist.

## Files Created/Modified

### Created
- `crates/anvilml-registry/src/db.rs` — SQLite pool creation, connection URL parsing, schema initialization via `sqlx::Executor::execute()`
- `crates/anvilml-registry/migrations/001_models.sql` — SQL migration: `models` table + indexes

### Modified
- `crates/anvilml-registry/Cargo.toml` — Added `sqlx`, `tracing` deps; `tokio` dev-dep for async tests
- `crates/anvilml-registry/src/lib.rs` — Added `pub mod db;`
- `crates/anvilml-registry/src/registry.rs` — Full rewrite: async `ModelRegistry` with SQLite backend

## Implementation Details

### Database Layer (`db.rs`)
- **Pool creation**: `create_pool(url)` parses URL → creates `SqlitePool` → runs schema init
- **URL parsing**: Supports `sqlite::memory:`, `sqlite:<path>`, and bare file paths
- **Schema init**: Uses `sqlx::Executor::execute()` on a pooled connection within a transaction (avoids sqlx compile-time validation issues when `.sqlx` cache is unavailable)
- **Table schema**: `models` table with `id` PK, `path` UNIQUE, indexes on `kind` and `dtype_hint`

### Registry (`registry.rs`)
- **`new(pool)`**: Wraps `SqlitePool` in `Arc`
- **`scan_and_persist(dirs)`**: Scans dirs → upserts each model via `INSERT OR REPLACE` (idempotent)
- **`list(kind)`**: SELECT with optional `WHERE kind = ?` filter
- **`get(id)`**: Single row lookup by ID
- **`count()`**: `SELECT COUNT(*)`
- **`estimate_vram_mib(size, dtype)`**: Delegates to scanner heuristic

### String Conversion Helpers
- `kind_to_str` / `str_to_kind`: ModelKind ↔ SQLite TEXT (lowercase)
- `dtype_to_str` / `str_to_dtype`: DType ↔ SQLite TEXT (snake_case)

## Test Results

**23 tests passed, 0 failed:**

### Scanner tests (14) — unchanged from P1-B1
### Registry tests (9):
- `test_scan_and_persist` — discovers and persists 1 model
- `test_list_returns_all` — returns all persisted models
- `test_list_filters_by_kind` — filters by ModelKind correctly
- `test_get_by_id` — correct lookup by ID
- `test_get_unknown_id` — returns None for unknown
- `test_scan_idempotent` — re-scan produces no duplicates
- `test_count` — count starts at 0, increments correctly
- `test_estimate_vram` — VRAM delegation verified
- `test_multiple_models` — multiple dirs produce correct count

## Acceptance Criteria

- [x] `cargo test -p anvilml-registry --lib` exits 0, ≥5 tests pass (23 passed)
- [x] `cargo clippy -p anvilml-registry -- -D warnings` exits 0
- [x] Git commit + push completed (99bef4f on backend main)
- [x] Scanner integration preserved — scan_and_persist bridges scanner → SQLite
