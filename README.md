# SindriStudio

**SindriStudio** is an AI image generation platform with a modular backend and a React-based frontend test panel.

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│  sindristudio  (launcher binary)                        │
│  ├── Starts AnvilML HTTP server                         │
│  ├── Spawns Python inference workers                    │
│  └── Opens browser to BloomeryUI                        │
└─────────────────────────────────────────────────────────┘
                            │
              ┌─────────────▼──────────────┐
              │  AnvilML (Rust backend)     │◄──── REST + WebSocket ────► BloomeryUI
              │  axum HTTP server :8188     │                             (React SPA)
              │  ├── Scheduler              │
              │  ├── Worker pool            │
              │  ├── Model registry         │
              │  └── Hardware detector      │
              └─────────────────────────────┘
```

## Components

| Component | Repository | Description |
|-----------|-----------|-------------|
| **AnvilML** | [backend/](backend/) | Rust backend — HTTP server, job scheduler, worker pool, hardware detection |
| **BloomeryUI** | [frontend/](frontend/) | React SPA — developer test panel for monitoring and controlling the backend |

## Getting Started

1. Clone the repository (including submodules):
   ```bash
   git clone --recurse-submodules https://github.com/DrywFiltiarn/SindriStudio.git
   cd SindriStudio
   ```

2. Initialize submodules (if cloned without `--recurse-submodules`):
   ```bash
   git submodule update --init --recursive
   ```

3. Start the backend:
   ```bash
   cd backend
   cargo build --release
   ./target/release/sindristudio
   ```

4. The launcher opens BloomeryUI in your default browser at `http://127.0.0.1:8188`.

## Hardware Support

- NVIDIA GPU (CUDA)
- AMD GPU (ROCm)
- Intel GPU (IPEX)
- CPU (fallback)

## License

Private / Proprietary — SindriStudio Project
