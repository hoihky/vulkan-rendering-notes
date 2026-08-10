# SparkDemo Mode Catalog

> **Build-Along Milestone:** Run mapped demo for every completed part with clean validation capture.

**You will learn:**

- Pick the correct demo to validate each book part milestone.
- Know which subsystems each demo stresses.
- Use demos as smoke tests before renderer refactors.

## 1. The Problem We Are Solving

Spark ships **twenty demo modes** in one executable — visual ground truth while building your own engine. Without a catalog you launch Basic 3D for every chapter and conclude "shadows are broken" when shadows were never enabled in that mode.

This appendix maps **demo → book part → subsystems** so you run the right smoke test after each chapter.


Appendix chapter 04 supports active development — bookmark it. When stuck mid-chapter on Parts XV–XVI, grep appendix first before re-reading entire prior part. Keep local notes column for your renamed types alongside Spark names.

Schedule 30-minute appendix refresh after any PR touching `SceneSubmit`, descriptor layouts, or `RecreateSwapchain`.


Run demo 17 once before reading the stress matrix — having seen Time of Day visually anchors which subsystems the catalog claims to exercise. Demo 1 alone never shows SSAO or CSM.

## 2. Core Concepts

### Demos as milestone tests

Each book part maps to SparkDemo modes — after completing exercises, run the mapped demo with validation clean. Demos are **integration tests**, not shipping products (Part XVI templates differ).

Failure mode: finishing Part IX shadows but testing Basic 3D — feature not enabled there yet.

### Launcher catalog

Shell menu lists 20 labeled modes — numeric select at startup:

*Demo launcher menu strings*

```cpp
// src/spark/demo/SparkShellDemo.cpp (lines 38–59)
static constexpr const char* kLauncherDemoLabels[] = {
        "1  - Basic 3D scene",
        "2  - Sky: box, dome, plane",
        "3  - Particles",
        "4  - Terrain",
        "5  - Character (1st / 3rd person)",
        "6  - 2D platformer",
        "7  - 2D Maze",
        "8  - 3D maze",
        "9  - 3D physics",
        "10 - 3D scene editor",
        "11 - Tetris",
        "12 - Match-3",
        "13 - Space Invaders",
        "14 - 3D steering",
        "15 - Toon/cel shading",
        "16 - Material ball",
        "17 - Time of day",
        "18 - Farming RPG render layers",
        "19 - Tilemap layers, animation & pathfinding",
        "20 - Dear ImGui tools (docking)",
};
```

### Part → demo mapping

| Part focus | Suggested demo |
|------------|----------------|
| I–II Bootstrap, mesh | 1 Basic 3D |
| IV Materials | 16 Material ball |
| VIII Particles | 3 Particles |
| IX–X Lighting/post | 17 Time of day |
| XI 2D | 6 Platformer, 19 Tilemap |
| XII Terrain/cull | 4 Terrain |
| XIII Editor | 10 Scene editor |
| XIV Physics | 9 3D physics |
| XV AI | 14 3D steering |
| XVI Hardening | Any + resize stress |

Keep table in repo README — onboarding shortcut.

### SPIR-V rebuild discipline

After GLSL edit:

```bash
cmake --build build --target SparkShaders -j
```

Stale SPIR-V causes "shader looks unchanged" confusion — always rebuild shaders target before blaming C++.

Set `SPARK_SHADER_SPV_DIR` when running binary outside IDE preset.

### RenderDoc regression captures

Save one capture per part milestone tagged `part09_time_of_day.rdc`. Human diff pass order when refactoring tonemap — faster than pixel tests for structural regressions.

Compare event tree: Shadow → HDR → SSAO → Tonemap → Present.

### DemoMode enum

`include/spark/demo/DemoMode.hpp` lists programmatic ids — shell maps numbers to factory functions. Grep when adding demo 21 — update labels array and enum together.

### Demo 17 Time of day scope

Demo 17 integrates directional shadows, HDR, tonemap, SSAO, time-of-day sun — run after Parts IX and X complete. Capturing RenderDoc here validates full frame graph before Part XVI hardening.

Demo 1 Basic 3D lacks punctual shadows and post — do not use for SSAO regression.

### Shader rebuild CI hook

Add CI step `cmake --build build --target SparkShaders` before running headless demo smoke — stale SPIR-V causes false failures that look like C++ regressions.

### Headless / CI demo run

Some demos require GPU window — CI may run NullSoundOutput path with SwiftShader or skip GPU tests on macOS agents. Document which demos are manual-only (Scene editor).

Automate: build SparkShaders, launch demo N with `--exit-after 300` if fork adds flag for smoke.

### Performance baseline demos

Demo 4 Terrain stresses cull + many draws; Demo 17 stresses post stack — pick baselines matching change area when refactoring renderer internals.

### Full demo roster (subsystem focus)

| # | Mode | Stress |
|---|------|--------|
| 1 | Basic 3D | Mesh + camera + forward pass |
| 2 | Sky box/dome/plane | Sky draw modes |
| 3 | Particles | Particle pass, blending |
| 4 | Terrain | TerrainComponent, culling |
| 5 | Character camera | 3rd person rig |
| 6 | Platformer | 2D physics, sprites, audio |
| 7 | 2D maze | Top-down 2D |
| 8 | 3D maze | 3D navigation |
| 9 | 3D physics | Rigidbody3D |
| 10 | Scene editor | Serialize, viewport |
| 11 | Tetris | 2D arcade + UI |
| 12 | Match-3 | Grid gameplay |
| 13 | Space Invaders | Shooter loop |
| 14 | 3D steering | AI agents |
| 15 | Toon | Cel shading model |
| 16 | Material ball | PBR + IBL |
| 17 | Time of day | Shadow + HDR + SSAO |
| 18 | Farming RPG layers | Sorting groups |
| 19 | Tilemap advanced | Anim + pathfinding |
| 20 | ImGui docking | Debug UI |

Use this table when a teammate asks "which demo shows X?" — answer from stress column, not guesswork.

### Complete demo catalog with stress matrix

| # | Label | Book part | Primary stress | Secondary | Validation focus |
|---|-------|-----------|----------------|-----------|------------------|
| 1 | Basic 3D | I–III | Swapchain, mesh draw | Uniform ring | Frame sync |
| 2 | Textured mesh | IV | Texture upload | Descriptor array | Layout transitions |
| 3 | PBR materials | VII | Multi-map PBR | Normal mapping | sRGB |
| 4 | ThreeDDemo | V–VIII | Full forward scene | Particles | Draw batching |
| 5 | Shadow showcase | IX | CSM + punctual | PCF | Shadow pass order |
| 6 | Platformer 2D | XI | Sprites + composite | Audio BGM | Sort keys |
| 7 | Render layers 2D | XI | Lighting modes | Instancing | SSBO |
| 8 | Sprite animation | XI | Flipbook UV | FSM | Atlas inset |
| 9 | Tilemap | XI | Chunk cull | Multi-layer | Visible tile count |
| 10 | Scene editor | XIII | Save/load | Viewport resize | Async Pump |
| 11 | Tetris | XIV | UI + 2D physics | Screen UI | Post-tonemap UI |
| 12 | Match-3 | XIV | Grid logic | Input | — |
| 13 | Space Invaders | XIV | Arcade loop | Collision | — |
| 14 | 3D steering | XV | AI agents | Blackboard | CPU only |
| 15 | Toon | VII | Cel shading | Material mode | Shader branch |
| 16 | Material ball | VII | IBL | Environment | Cubemap |
| 17 | Time of day | X | HDR + SSAO + TOD | Shadows | Full stack |
| 18 | Farming RPG | XI | Sorting groups | Layer policy | Stable sort |
| 19 | Tilemap advanced | XI | Anim tiles | Pathfinding | Chunk cull |
| 20 | ImGui docking | XIII | Editor UI | Multi-viewport | Input routing |

**Weekly smoke schedule (solo dev):** Mon demo 1, Wed demo 6 or 9, Fri demo 17. Full matrix before merge to main renderer branch.

### Demo dependency graph

Demos share engine code but stress different **SceneSubmit fill paths**:

```text
SparkShellDemo (menu)
  ├─ Basic3DDemo ──────────── minimal draws, no shadows
  ├─ ThreeDDemo ───────────── full forward + particles
  ├─ Platformer2DDemo ─────── sprites + audio + 2D physics
  ├─ TerrainDemo ──────────── terrain + frustum cull
  ├─ SceneEditor3DDemo ────── async scene + viewport resize
  └─ TimeOfDayDemo ────────── shadows + HDR + SSAO + TOD
```

When refactoring `VulkanRenderer`, run **lowest** and **highest** demo in dependency graph — regression in Basic3D often breaks all; regression only in TimeOfDay isolates post stack.

**Shared code path grep targets:**

```bash
rg 'RecordSceneCommandBuffer' src/spark/demo/
rg 'FillSceneFromWorld' src/spark/demo/
```

Demo-specific code should stay in `OnUpdate` / spawn — not fork renderer record path.

Fork for shipping: copy one demo's `Game` class, strip shell menu, keep SparkDemo in CI for regression.

### SPIR-V rebuild and stale shader bugs

Symptom: "I edited scene.frag but output unchanged" — stale SPIR-V in build tree.

**Discipline:**

```bash
cmake --build build --target SparkShaders
# or full rebuild if CMake dep chain broken
```

Engine links `SparkShaders` custom target — student forks that copy `.spv` manually forget dependency edge. Add to CI:

```bash
ninja SparkShaders && git diff --exit-code build/shaders/
```

Shader change without SPIR-V rebuild shows as **old behavior** with no compile error — especially insidious for uniform layout edits (garbage until validation catches mismatch).

Document in appendix demo catalog: after Part III shader exercises, always rebuild shaders before launching demo for that part's milestone.

### Keyboard launcher quick reference

SparkShellDemo maps numeric keys to demos — appendix catalog numeric column matches launcher:

```text
Keys 1–9, 0, then letters per shell implementation
Menu displays kLauncherDemoLabels[i] for human name
```

**Fast validation session (15 min):**

```text
1 → bootstrap OK
6 → 2D composite + audio
9 → tilemap cull
10 → editor save/load (if Part XIII done)
17 → full stack
exit → zero validation leaks
```

Log demo index and validation result in `docs/demo-smoke-log.md` — one line per session. Trend of failures on demo 17 only signals post-stack regression; failures on demo 1 signal core renderer breakage.

Automating keyboard input for smoke optional — `--demo 17 --duration 30` CLI flag cleaner than scripted keypress if you add it to fork.

### Demo-specific known validation quirks

Document known benign validation messages per demo to avoid false panic:

| Demo | Quirk | Action |
|------|-------|--------|
| ImGui docking 20 | Many small dynamic VBs | Expected high buffer count |
| Scene editor 10 | Viewport resize during edit | Recreate must stay clean |
| Particles heavy 4 | Large SSBO | Verify pool size once |
| Time of day 17 | Many passes | Long RenderDoc capture |

If validation error appears in demo 17 but not demo 1, bisect which pass addition introduced it — shadow, SSAO, or tonemap — using demo dependency graph from this appendix.

Keep `docs/demo-validation-notes.md` with VUID, demo #, status (fixed/wontfix), and Spark commit hash — appendix living doc.

### Forking demo for MyGame template

Part XVI game template typically forks **ThreeDDemo** or **Platformer2DDemo**:

```text
Copy Demo class → MyGame.cpp
Remove DemoMode enum switch case bloat
Keep single OnAttach scene setup
Wire quality preset JSON load in OnAttach
Replace shell menu entry with direct MyGame launch target
```

Retain original demo source in repo for CI — MyGame and ThreeDDemo share `VulkanRenderer` static library; diff should be gameplay layer only.

Catalog column "Primary stress" tells which demo to fork — do not fork Tetris if shipping 3D RPG (wrong submit fill path for regression).

### Part-to-demo milestone schedule

Use this **reading order smoke schedule** after completing each book part — run the listed demo before starting the next part:

| After part | Demo # | Name | Confirms |
|------------|--------|------|----------|
| I–III | 1 | Basic 3D | Swapchain, mesh, shaders |
| IV | 2 | Textured mesh | Upload, descriptors |
| V | 4 | ThreeDDemo | Camera, forward scene |
| VII | 16 | Material ball | PBR maps |
| IX | 5 | Shadow showcase | CSM + punctual |
| X | 17 | Time of day | HDR, SSAO, tonemap |
| XI | 6, 9 | Platformer, Tilemap | 2D composite, cull |
| XII | Terrain | TerrainDemo | Heightfield, frustum cull |
| XIII | 10 | Scene editor | Save/load, async |
| XIV | 11–13 | Arcade demos | Physics 2D |
| XV | 6, 14 | Platformer, Steering | Audio, AI |
| XVI | 17 + MyGame | Full stack + retail | Hardening |

Skipping a row and jumping ahead produces false "book broken" reports — the catalog is the curriculum's acceptance test, not optional reading. Log each run in regression notebook with date and git SHA.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Part → demo mapping table

**Concept:** Demos as milestone tests

**Decision:** Part → demo mapping table

**Why:** Prevents testing wrong subsystem — teaching team uses same table in code reviews.

**In Spark:** Appendix + course README.
### SparkShellDemo kLauncherDemoLabels

**Concept:** Launcher catalog

**Decision:** SparkShellDemo kLauncherDemoLabels

**Why:** Single array documents human names — grep-friendly for book writers.

**In Spark:** `SparkShellDemo.cpp` lines 38–59.
### SparkShaders CMake target

**Concept:** SPIR-V rebuild discipline

**Decision:** SparkShaders CMake target

**Why:** Engine depends on shaders — reduces stale SPIR-V in student builds.

**In Spark:** Part III CMake chapter.
### Tagged by part number

**Concept:** RenderDoc regression captures

**Decision:** Tagged by part number

**Why:** Lightweight visual CI substitute for solo developers.

**In Spark:** Part XVI profiling culture.
### Fork one demo in Part XVI

**Concept:** Demo vs shipping

**Decision:** Fork one demo in Part XVI

**Why:** Catalog remains reference; shipping removes selector.

**In Spark:** Game Templates chapter.
### Weekly smoke schedule for solo dev

**Concept:** Complete demo catalog with stress matrix

**Decision:** Weekly smoke schedule for solo dev

**Why:** Demo 1 for shadows gives false negative — matrix maps part to demo.

**In Spark:** SparkShellDemo.cpp kLauncherDemoLabels.

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `src/spark/demo/SparkShellDemo.cpp`
- `include/spark/demo/DemoMode.hpp`

### Step 1 — Build and launch shell

```bash
cmake --build build -j
./build/SparkDemo
```

Print menu — match labels to your completed part number.
### Step 2 — Run mapped demo for latest part

If you finished Part X, launch **17 - Time of day** — confirm shadows + HDR + tonemap + SSAO visible.
### Step 3 — Validation clean run

Enable layers, run demo 60 s, exit — zero errors. Log any VUID to issue tracker before continuing book.
### Step 4 — Capture RenderDoc baseline

Save `.rdc` with part tag — note pass count and average GPU duration from overlay.
### Step 5 — Part XI 2D milestone

Launch **6 Platformer** or **19 Tilemap** — HDR composite with sprites and tilemaps.
### Step 6 — Part XIII editor milestone

Launch **10 Scene editor** — save/load round-trip scene file from Chapter 1 exercises.
### Step 7 — Compare two demos' shared code

Pick demos 1 and 17 — identify shared `VulkanRenderer` path vs demo-specific SceneSubmit fill.
### Step 8 — Maintain regression notebook

Markdown table: part, demo #, capture path, date, notes — update after each renderer refactor.

## 5. How This Connects to the Rest of Your Engine

**Component reference** lists ECS types demos spawn. **Profiling** uses these workloads for baselines. **Game templates** fork source demos — catalog stays for regression.


Part **XVI** profiler baselines reference demo indices from this catalog — JSON field `demoIndex` must match table column #. Part **XVI** ch05 retail build still runs SparkDemo in CI even if shipping MyGame only.

## 6. Pitfalls and Debugging

- **Testing shadows in demo 1 before Part IX** — false negative.
- **Stale SPIR-V** — rebuild SparkShaders after shader edit.
- **Wrong demo for subsystem** — use mapping table.
- **Skipping validation on demo run** — demos catch lifetime bugs early.
- **Confusing shell with shipping** — menu is test harness only.


- **Running demo 1 for Part X milestone** — false negative on SSAO/HDR; use demo 17.
- **Skipping weekly smoke schedule** — regressions accumulate until demo 17 fails.

## 7. Exercises

1. Run stress matrix row for latest completed part; log validation status.
2. Add one line to regression notebook: part, demo #, capture path, date.
3. Compare shared VulkanRenderer path between demos 4 and 17 via grep.

## Summary

SparkDemo modes validate book milestones from Basic 3D through editor and physics. Use launcher catalog, shader rebuild, and RenderDoc captures as regression harness. Demo stress matrix, dependency graph, SPIR-V discipline, and 15-minute smoke session support daily renderer development.

---

Next: [Component Quick Reference](05-component-ref.md).
