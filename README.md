# MYA 3D Editor

An original, Maya-inspired 3D content creation application, built in Python with PySide6/Qt6 and a real-time OpenGL viewport.

MYA 3D Editor is **not** affiliated with or endorsed by Autodesk. It is an independent application that follows the general *workflow* conventions of professional 3D DCC (Digital Content Creation) tools (outliner / viewport / properties / timeline layout) without copying any copyrighted visual assets, branding, or code.

## Current Phase: Phase 2 — Real-Time 3D Viewport

Phase 1 delivered the professional editor **shell**. Phase 2 replaces the placeholder viewport with a real, modern OpenGL 3.3 viewport: real GPU-rendered geometry, a navigable editor camera, a world-space grid, basic lighting, GPU object picking, and a first working move interaction — all built on top of, and without breaking, the Phase 1 scene model, Outliner, Properties, Timeline, selection system, and undo/redo command stack.

## Features Implemented

### Phase 1 (editor shell — unchanged, still working)
- Main window with dark theme, saved window geometry/dock layout, resizable panels
- Menu bar: File, Edit, Create, Select, Modify, Display, Windows, Help
- Main toolbar (shared `QAction`s): New/Open/Save, Undo/Redo, Select/Move/Rotate/Scale
- Outliner (left dock): live scene tree, inline rename, multi-select, context menu
- Properties (right dock): Name/Type/Position/Rotation/Scale for the selection, edits routed through `TransformObjectCommand`/`RenameObjectCommand`
- Timeline (bottom dock): frame range, scrubber, transport buttons (frame data only — no animation playback yet)
- Status bar: selection, active tool, current frame
- Scene data model (`core/`): `Scene`, `SceneObject`, `Transform`/`Vector3` — framework-agnostic, no Qt imports
- Central `SelectionManager` and `CommandStack` (undo/redo for create/delete/rename/duplicate/transform)
- Object creation (Group, Null, Camera, Light, Cube, Sphere, Cylinder, Plane), rename, delete, duplicate (new ID + independent `Transform`)
- Versioned JSON `.mya` scene serializer

### Phase 2 (new)
- **Real OpenGL 3.3 core-profile viewport** (`ui/viewport.py`, `ViewportGLWidget`, a `QOpenGLWidget`) — replaces the Phase 1 placeholder text entirely. Creates a real GL context (`QSurfaceFormat`, depth 24 + stencil 8 + 4x MSAA), initializes once, clears/renders every frame, and releases all GPU resources safely on context teardown.
- **`OpenGLRenderer`** (`graphics/opengl_renderer.py`) — a concrete implementation of the existing `RendererInterface` (`initialize()` / `resize()` / `render(scene, camera)` / `shutdown()`). Owns all shader programs and a cache of GPU meshes; contains **no scene-management or selection logic** — it only reads `Scene`/`SceneObject` data to decide what to draw.
- **Modern OpenGL only** — VAOs, VBOs, an EBO per mesh, GLSL 330 core vertex/fragment shaders, uniform matrices, depth testing, back-face culling. No `glBegin()`/`glEnd()` anywhere.
- **Shader system** (`graphics/shaders/`): `basic.vert`/`basic.frag` (lit objects), `grid.vert`/`grid.frag` (grid & axis lines), `picking.frag` (offscreen ID-color picking pass, reuses `basic.vert`). Compilation/link failures raise `ShaderCompilationError` with the shader name and the driver's own log — never silently ignored.
- **3D math library** (`graphics/math3d.py`) — pure NumPy: translation/scale/rotation/model matrices, a right-handed `look_at` view matrix, a perspective projection matrix, screen↔NDC conversion, and ray/plane intersection helpers, fully unit-tested without a GPU.
- **Editor camera** (`graphics/camera.py`, `EditorCamera`) — an orbit camera (target + distance + yaw/pitch, pitch clamped to ±89° so it can never flip), with `get_view_matrix()` / `get_projection_matrix()`, `orbit()`/`pan()`/`zoom()`/`frame_target()`/`reset()`. Explicitly separate from any in-scene `Camera` `SceneObject` — see "Editor Camera vs. Scene Camera" below.
- **Mesh abstraction** (`graphics/mesh.py`) — `MeshData` (plain NumPy positions/normals/indices, GPU-independent, fully testable) vs. `MeshGPU` (owns the actual VAO/VBO/EBO, uploaded once, drawn every frame, released on shutdown). `SceneObject` never touches OpenGL directly.
- **Procedural primitives** (`graphics/primitives.py`) — real vertex/normal/index geometry for Cube (24 verts, flat face normals), Plane (configurable segments, +Y normal), UV Sphere (configurable rings/segments, smooth normals), and Cylinder (side wall + top/bottom caps, configurable segments). Every primitive is validated (`MeshData.is_valid()`) and unit-tested.
- **Create > Cube/Sphere/Cylinder/Plane now actually render** — each creates a real `SceneObject` (type `mesh`, `metadata={"primitive": ...}`), added to `Scene`/Outliner/selection/Properties exactly as in Phase 1, and now also rendered in the viewport at its `Transform`. Multiple objects at different positions/rotations/scales render correctly and independently; primitive mesh GPU buffers are cached and shared across every object using that primitive (never rebuilt per-object or per-frame).
- **World grid + axes** — a real world-space X/Z ground grid (not a 2D image), with colored X/Y/Z axis lines through the origin, drawn with its own shader, toggled by **Display > Grid** (hiding the grid never hides mesh objects).
- **Basic real-time lighting** — one directional light computed in `basic.frag`: ambient + Lambertian diffuse + a small non-PBR specular highlight, using the world-space normal (transformed by the inverse-transpose of the model matrix, correct under non-uniform scale) and the base color from `SceneObject.metadata["color"]` (defaults to a neutral gray).
- **Viewport navigation** — orbit (Alt+Left-drag or Right-drag), pan (Middle-drag), zoom (mouse wheel); see "Mouse Controls" below. Smooth, and the camera can never flip through the poles.
- **GPU color-picking selection** — clicking an object renders every mesh into an offscreen framebuffer with a unique flat ID color per object, reads back the clicked pixel, and maps it to a `SceneObject` id — not a screen-distance heuristic. Clicking empty space deselects. Selection flows through the same central `SelectionManager` in every direction (Viewport → SelectionManager → Outliner + Properties, and Outliner → SelectionManager → Viewport + Properties); no UI widget talks to another directly.
- **Selection visualization** — selected objects are tinted (blended toward a highlight color in the shader) without permanently altering `metadata["color"]`; this is pure render state (`OpenGLRenderer.selected_ids`), never written back into scene data.
- **Basic Move interaction** — with the Move tool active and an object selected, dragging in the viewport moves it across a camera-facing plane through the object (via ray/plane intersection); hold **X**, **Y**, or **Z** during the drag to constrain to that world axis. The whole drag becomes a single undoable `TransformObjectCommand` on release (not per-pixel undo spam). Rotate/Scale tools remain selectable (toolbar + Modify menu still work) but a full manipulator/gizmo for them is explicitly future work.
- **Frame Selected** (Select menu / **F** key while the viewport has focus) — moves the camera to frame the selected object using its actual mesh bounding radius and `Transform.scale`.
- **Reset View** (Display menu) — restores the default camera target/distance/angles.
- **Viewport resize handling** — `resizeGL` updates the GL viewport, the picking framebuffer, and the camera's aspect ratio/projection matrix together, so there is never stretching or a stale depth buffer after a resize.
- **`.mya` serialization, updated carefully** — mesh objects' `metadata["primitive"]` already round-trips through the existing generic `metadata` dict (no format version bump needed: the schema shape didn't change). **Existing Phase 1 `.mya` files load unchanged.** No raw GPU buffers or OpenGL state are ever serialized — only logical scene data.

## Features Intentionally NOT Implemented (Phase 2)

Per the Phase 2 specification, these remain explicitly out of scope:

- PBR materials, texture mapping, a UV editor, or a node/shader graph
- Vertex/edge/face editing, extrude, bevel, sculpting
- Rigging, skinning, skeletons
- A real animation/keyframe system (the Timeline UI exists; frames don't drive rendering yet)
- Physics, particles, fluid/hair simulation
- A full professional Rotate/Scale manipulator gizmo (Move has a basic axis-constrained drag; Rotate/Scale tools are selectable but not yet interactive in the viewport)
- Scene-Camera rendering / camera switching (a `Camera` `SceneObject` exists in the scene graph, but Phase 2 always renders through the separate editor camera — see below)
- A Python scripting console, plugin system, or import/export pipeline
- Cut/Copy/Paste (still menu stubs, as in Phase 1)

These are deferred to future phases so Phase 2 stays focused on a correct, working real-time rendering pipeline.

## Editor Camera vs. Scene Camera

`graphics/camera.py::EditorCamera` is a **UI navigation aid** — like the viewport camera in any DCC application. It is never serialized into `.mya` files and has no `SceneObject` of its own.

A `Camera`-type `SceneObject` (created via **Create > Camera**) is a real, logical scene object with its own `Transform`, visible in the Outliner and Properties, and saved/loaded with the scene — but in Phase 2 the viewport always renders through the separate `EditorCamera`, not through any scene Camera object. Rendering *through* a scene camera (camera switching) is future work.

## Coordinate Convention

Documented once, authoritatively, in `graphics/math3d.py` (module docstring) and followed consistently everywhere (camera, meshes, transforms):

- Right-handed coordinate system.
- **X** = left(−) / right(+)
- **Y** = down(−) / up(+) — **Y is up**
- **Z** = into-screen(−) / out-of-screen(+) — the camera looks down its own local **−Z**
- Units are arbitrary "scene units" (treat as meters by convention)
- Rotations are Euler angles in **degrees**, applied X → Y → Z (model matrix = `T · Rz · Ry · Rx · S`, column-vector math: `v' = M · v`)

## Mouse Controls

| Action | Control |
|---|---|
| Select object | Left-click (no drag) on it; click empty space to deselect |
| Orbit | **Alt + Left-drag**, or **Right-drag** |
| Pan | **Middle-drag** |
| Zoom | Mouse wheel |
| Move selected object | Left-drag with the **Move** tool active and an object selected |
| Constrain move to an axis | Hold **X**, **Y**, or **Z** while dragging (press again to release the lock) |

## Keyboard Shortcuts

Global (safe everywhere — modifier-based, so they never interfere with typing):

| Shortcut | Action |
|---|---|
| Ctrl+N / Ctrl+O / Ctrl+S / Ctrl+Shift+S | New / Open / Save / Save As |
| Ctrl+Z / Ctrl+Shift+Z | Undo / Redo |
| Ctrl+D | Duplicate selected |
| Ctrl+Q | Exit |

Viewport-only (single letters — these are handled **inside the viewport widget itself** and only fire while the 3D viewport has keyboard focus, so renaming an object in the Outliner or typing in the Properties Name field is never interrupted):

| Key | Action |
|---|---|
| Q / W / E / R | Select / Move / Rotate / Scale tool |
| Delete / Backspace | Delete selected object(s) |
| F | Frame Selected |

## Grid Visibility

**Display > Grid** toggles the world-space grid and axis lines. Hiding the grid never hides mesh objects — they remain visible and correctly lit/occluded.

## Current Limitations

- Rotate/Scale tools are selectable but have no interactive viewport manipulator yet (only Move does).
- The viewport's "View" preset combo box (Top/Front/Side/...) sets a perspective viewing *angle*, not a true orthographic projection — full ortho views are future work.
- No scene-Camera rendering/switching yet (see "Editor Camera vs. Scene Camera" above).
- Wireframe mode is a real OpenGL polygon mode (`GL_LINE`), not a fake overlay, but has not been tested against every GPU driver.
- This environment could not create a real OpenGL context to visually verify rendering — see "Testing" below for exactly what was and wasn't verified.

## Requirements

- Python 3.12+
- PySide6 (Qt6) — window, widgets, `QOpenGLWidget`
- PyOpenGL — Python bindings for the OpenGL calls the renderer makes
- NumPy — all matrix/vector math (required; a hand-rolled matrix library would duplicate well-tested, fast code for no benefit)
- pytest (for running tests)

See `requirements.txt`.

## Installation

```bash
# from the project root (the directory containing this README)
python -m venv .venv
```

Windows:
```bash
.venv\Scripts\activate
```

macOS/Linux:
```bash
source .venv/bin/activate
```

Then, on any platform:
```bash
pip install -r requirements.txt
```

## Running the Application

```bash
python main.py
```

On launch you should see the main window with menu bar, toolbar, Outliner (left), a real 3D Viewport (center) showing a ground grid and axes from the default camera angle, Properties (right), Timeline (bottom), and status bar — with a default scene containing a Camera and a Light already in the Outliner (no default mesh is added; create one via **Create > Cube**, etc.).

## Running Tests

```bash
pytest
```

`tests/conftest.py` adds the project root to `sys.path` so tests run directly with `pytest` from the project root without an editable install. The suite is entirely CPU-side/unit-level (no GPU or Qt event loop required):

- `test_transform.py`, `test_scene.py`, `test_selection.py`, `test_commands.py`, `test_serializer.py` — Phase 1 core/editor/scene-I/O behavior (unchanged)
- `test_math3d.py` — matrix/vector math (translation, scale, rotation, model, look-at, perspective, screen↔NDC, ray/plane intersection)
- `test_camera.py` — `EditorCamera` orbit/pan/zoom/clamping/reset
- `test_primitives.py` — Cube/Sphere/Cylinder/Plane geometry validity, vertex/index counts, normals
- Mesh-primitive-metadata round-tripping and multi-object-transform tests added to `test_serializer.py`/`test_scene.py`

GPU-dependent code (`MeshGPU.upload()/draw()`, `ShaderProgram.compile_and_link()`, the full `OpenGLRenderer`, and all Qt widget code) is **not** covered by this test suite, since it requires a real OpenGL context and cannot be safely unit-tested headlessly — see "Testing" below for how to verify it manually.

## Project Architecture

```
mya3d/
├── main.py                     # Thin entry point: logging + QApplication + MainWindow
│
├── app/                        # Application bootstrap (unchanged from Phase 1)
│   ├── application.py
│   └── settings.py
│
├── core/                       # Framework-agnostic scene data model (unchanged from Phase 1)
│   ├── ids.py / object_types.py / transform.py / scene_object.py / scene.py / signal.py
│
├── editor/                     # Editor logic (unchanged from Phase 1)
│   ├── selection.py / tools.py / commands.py
│
├── ui/                         # PySide6 widgets
│   ├── main_window.py          # Assembly/wiring only — now also wires viewport <-> Grid/Wireframe/Reset View/Frame Selected
│   ├── viewport.py             # Phase 2: ViewportGLWidget (QOpenGLWidget) + ViewportWidget (toolbar wrapper)
│   ├── outliner.py / properties.py / timeline.py / toolbar.py / styles.py   # unchanged from Phase 1
│
├── sceneio/
│   └── scene_serializer.py     # Versioned JSON .mya save/load (unchanged shape; mesh metadata already round-trips)
│
├── graphics/                   # Phase 2: the real rendering pipeline
│   ├── renderer_interface.py   # RendererInterface ABC (render() now takes scene+camera) + NullRenderer
│   ├── opengl_renderer.py      # OpenGLRenderer: implements RendererInterface, owns shaders/mesh cache/picking FBO
│   ├── camera.py                # EditorCamera (orbit camera)
│   ├── mesh.py                  # MeshData (CPU) / MeshGPU (GPU buffers)
│   ├── primitives.py            # generate_cube/sphere/cylinder/plane
│   ├── math3d.py                 # Matrix/vector math, NumPy-based
│   └── shaders/
│       ├── basic.vert / basic.frag       # Lit mesh objects
│       ├── grid.vert / grid.frag         # Grid & axis lines
│       └── picking.frag                  # Offscreen ID-color picking pass
│
├── resources/icons/
│
├── tests/                      # pytest suite (CPU/unit only — no GPU/Qt required)
│   ├── conftest.py
│   ├── test_transform.py / test_scene.py / test_selection.py / test_commands.py / test_serializer.py
│   └── test_math3d.py / test_camera.py / test_primitives.py    # new in Phase 2
│
├── requirements.txt
├── .gitignore
└── README.md
```

**Design principles followed throughout (Phase 1 and Phase 2):**

- `core/` and `editor/` never import PySide6/Qt *or* OpenGL — plain Python, unit-testable headlessly.
- `graphics/` never imports Qt — `OpenGLRenderer` only knows shaders, GPU buffers, matrices, and draw calls. It reads `Scene`/`SceneObject` read-only and contains no selection/undo/UI logic; the viewport tells it what's selected each frame via `renderer.selected_ids`.
- `ui/` widgets read from `Scene`/`SelectionManager` and write through `CommandStack`; widgets never call each other directly — the Viewport, Outliner, and Properties dock all independently observe the same central `SelectionManager` and `Scene`.
- `MainWindow` only assembles and wires components.
- GPU resources (shaders, mesh buffers) are created exactly once — in `initialize()` or lazily-and-cached on first use — and released in `shutdown()`. Nothing is rebuilt per-frame.
- The `.mya` format is versioned; unsupported versions fail with a clear error rather than corrupting data. Existing Phase 1 files load unchanged in Phase 2.
- The `sceneio` package is named `sceneio`, not `io`, to avoid shadowing Python's built-in `io` module.

## Testing

This environment could not install PyOpenGL or PySide6 (no network access) and therefore could not create a real OpenGL context or run the Qt event loop. What was actually verified, and how, is reported precisely in the Phase 2 completion report delivered alongside this codebase — in short:

- **Verified by actually running the tests**: all CPU/unit tests (`core`, `editor`, `sceneio`, and the `graphics/math3d.py` + `graphics/camera.py` + `graphics/primitives.py` + `renderer_interface`/regression tests) — 127 test cases, 0 failures.
- **Verified by static review only** (syntax-checked, imports cross-checked, logic manually traced, but not executed): the OpenGL renderer, shader compilation, GPU mesh upload/draw, and all Qt widget/viewport code.
- **Not verified at all in this environment**: an actual running window, real GPU rendering, real mouse-driven navigation/picking, and the full manual acceptance workflow. These require a machine with a real GPU/driver and PySide6+PyOpenGL installed — see the exact command below.

To verify the real GUI/OpenGL behavior yourself:

```bash
pip install -r requirements.txt
python main.py
```

Then walk through: orbit/pan/zoom the camera, create a Cube/Sphere/Cylinder/Plane and confirm each renders as real lit 3D geometry at the correct position, click objects to select them (and confirm the Outliner/Properties update), drag with the Move tool active, toggle Display > Grid, save and reload a `.mya` file, and run `pytest`.

## Roadmap / Phase 3

Not started. Candidate scope, building on the Phase 2 rendering pipeline:

- Full Rotate/Scale transform gizmos (and a unified Move/Rotate/Scale manipulator) in the viewport
- Real animation/keyframe system layered on the existing `TimelineState`
- Scene-Camera rendering and camera switching
- True orthographic projection views
- Basic materials/textures (still short of full PBR)
- Vertex/edge/face-level mesh editing

Phase 3 has not been started; this repository currently contains Phase 1 + Phase 2 only.
