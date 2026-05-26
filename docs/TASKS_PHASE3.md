# TASKS_PHASE3.md — Phase 3: Node Graph + Advanced Inference
# High-level milestone definitions. Will be broken into atomic tasks
# when Phase 2 is complete and requirements are fully clear.
#
# Prerequisites: ALL Phase 2 tasks complete.

---

## PHASE 3 GOAL

Deliver a functional node graph editor in BloomeryUI, replacing the test panel
as the primary interface. Support LoRA, img2img, ControlNet. Improve inference
performance with TunableOp warming and multi-GPU routing.

---

## MILESTONE 3-A: Custom Node Graph Canvas

### 3-A1 — Canvas renderer foundation
Custom HTML5 Canvas/WebGL renderer. Pan, zoom, minimap.
No third-party node graph library — full control required.
Nodes render as styled rectangles with input/output ports.

### 3-A2 — Node connection system
Port-to-port connection by drag. Type checking on connect
(prevent connecting incompatible port types). Visual feedback.

### 3-A3 — Node interaction
Select single/multi. Move. Copy/paste. Undo/redo stack (50 levels).
Keyboard shortcuts (Delete, Ctrl+C, Ctrl+V, Ctrl+Z, Ctrl+Y).

### 3-A4 — Node widget system
Per-node input widgets: text field, number slider, dropdown, toggle.
Widgets embedded in node body. Values editable inline.

### 3-A5 — Workflow serialize/deserialize
Save workflow as JSON. Load from JSON. Auto-save on change.
Format is the same graph JSON the backend already accepts.

### 3-A6 — Real-time execution visualization
Highlight active node during inference. Show per-node progress.
Preview thumbnails displayed on sampler node output port.

---

## MILESTONE 3-B: Advanced Inference

### 3-B1 — LoRA support
LoRA loader node. Works with both ZiT and SDXL base models.
Strength slider. Multiple LoRA stacking.

### 3-B2 — Image-to-image
ImageInput node (upload from browser). img2img pipeline nodes
for ZiT and SDXL. Strength/denoise slider.

### 3-B3 — ControlNet support
ControlNet loader and apply nodes. Preprocessor nodes (Canny, Depth).

### 3-B4 — TunableOp cache warming
First-run tuning workflow for gfx1201. Cache saved to
`SindriStudio/.tunableop_cache_{gfx_arch}.csv`.
Subsequent runs load cache automatically.

---

## MILESTONE 3-C: Multi-GPU and Memory

### 3-C1 — Multi-GPU routing
Route different jobs to different GPU workers.
Device affinity hints in job settings.

### 3-C2 — VRAM eviction policy
Implement LRU eviction. When VRAM budget exceeded, evict
least-recently-used model before loading new one.

### 3-C3 — Look-ahead prefetch (activate)
Prefetch logic implemented in Phase 1 — wire it to actually
send PreloadModel messages based on queue look-ahead.

---

## MILESTONE 3-D: BloomeryUI Polish

### 3-D1 — Model browser sidebar
Panel listing available models by type. Drag model onto
canvas to create a loader node automatically.

### 3-D2 — Output gallery page
Dedicated gallery page. Filter by model, date, prompt.
Image comparison view (side by side).

### 3-D3 — Workflow library
Save named workflows. Browse and load saved workflows.
Export/import as .json files.
