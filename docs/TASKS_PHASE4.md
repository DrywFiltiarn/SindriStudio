# TASKS_PHASE4.md — Phase 4: MVP Completion
# High-level milestone definitions. Will be broken into atomic tasks
# when Phase 3 is complete.
#
# Prerequisites: ALL Phase 3 tasks complete.

---

## PHASE 4 GOAL

Deliver a polished, distributable MVP. Community node extensions,
authentication for remote server use, Apple Silicon support,
and a proper installer for Windows and Linux.

---

## MILESTONE 4-A: Extension System

### 4-A1 — Extension manifest format
`comfy-extension.toml`: declares node types, Python deps, frontend JS bundle.
Versioning, compatibility declarations.

### 4-A2 — Extension loader
Isolated Python subprocess per extension.
Dependency sandboxing via uv. Crash isolation.

### 4-A3 — Extension registry API
Install, remove, list, update extensions via REST.
BloomeryUI extension management panel.

### 4-A4 — ComfyUI node compatibility shim
Shim layer allowing existing ComfyUI custom nodes to run
with minimal modification. Covers top 20 most-used community nodes.

---

## MILESTONE 4-B: Distribution

### 4-B1 — Windows installer
NSIS or MSI installer. Bundles `sindristudio.exe` and Python venv.
PyTorch wheel selection (CUDA or ROCm) during install.
Auto-update via built-in updater.

### 4-B2 — Linux package
AppImage (portable, no install needed) + .deb + .rpm.
systemd service file for headless server operation.

### 4-B3 — Apple Silicon support
PyTorch MPS backend. `mps` device type in hardware detector.
MPS-specific memory model (unified memory — no VRAM/RAM split).
macOS DMG installer.

---

## MILESTONE 4-C: Authentication + Remote Access

### 4-C1 — API key authentication
Optional API key for local network protection.
`--no-auth` flag for local single-user mode (default).
Keys managed via CLI: `sindristudio key add/list/revoke`.

### 4-C2 — JWT session tokens
For multi-user remote deployments.
Rate limiting per key. Request logging.

### 4-C3 — BloomeryUI remote server
BloomeryUI can connect to remote AnvilML instances.
Server URL + API key entry in connection panel.
HTTPS support (self-signed cert generation via CLI).

---

## MILESTONE 4-D: Hardening + Community

### 4-D1 — Security audit
IPC boundary review. Extension sandboxing verification.
Local network exposure hardening.

### 4-D2 — Performance audit
Profile full generation pipeline. Optimize IPC overhead.
Batch WebSocket messages. Benchmark suite across hardware targets.

### 4-D3 — Documentation
Auto-generated API reference from OpenAPI spec.
Node SDK developer guide.
Extension development guide.
User manual (built into BloomeryUI as help panel).

### 4-D4 — Public beta
Deploy docs site. Community feedback channel.
Bug triage process. v1.0 release criteria defined.
