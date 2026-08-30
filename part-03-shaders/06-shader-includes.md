# Shader Includes and Shared Code

> **Build-Along Milestone:** You can draw the include graph for `scene.frag` and explain why editing `clustered_lights.glsl` triggers rebuild of every dependent SPIR-V.

**You will learn:**

- Use `#include` with `GL_GOOGLE_include_directive` in Vulkan GLSL.
- Organize shared UBO, lighting, and color-space code across passes.
- Wire CMake `DEPENDS` so header edits invalidate the right `.spv` files.

## 1. The Problem We Are Solving

Real shaders outgrow single files quickly. Shadow passes need the same UBO layout as the scene pass. Fragment lighting shares BRDF helpers with post-processing color transforms. Copy-pasting `struct Light` into five files guarantees drift within a week.

Spark uses **GLSL includes** with a flat `shaders/` directory and CMake dependency tracking. This chapter covers the include mechanism, Spark's file roles, and build hygiene — the glue that keeps a multi-pass renderer maintainable.

### Why split shaders into multiple files? (beginner)

Imagine `scene.frag` grew to 5,000 lines — lighting, shadows, sky, glTF extensions, fog, all in one file. Changing sun color would mean scrolling past particle math. **Includes** let you organize like C++ headers:

```text
scene.frag
  #include "scene_ubo.glsl"      ← camera + lights layout (must match C++)
  #include "ibl.glsl"            ← environment reflections
  #include "clustered_lights.glsl" ← punctual light loops
  #include "gltf_pbr_extensions.glsl"
```

Each `.glsl` file is still compiled **together** into one SPIR-V module — `#include` is textual paste at compile time, not a runtime link.

**Rules for beginners:**

1. **Shared layouts live in includes** — `scene_ubo.glsl` must match `SceneUniformGpu` byte-for-byte.
2. **Edit an include → rebuild dependents** — CMake tracks `DEPENDS` so `scene.frag.spv` rebuilds when `clustered_lights.glsl` changes.
3. **No circular includes** — keep a DAG: ubo → lighting → main shader.

If glslang reports "undefined identifier `ubo`", an include is missing or in wrong order.

## 1b. Concept Deep Dive — scene Pass Include Graph and CMake DEPENDS

### scene.vert — minimal include tree

The vertex stage only needs frame and per-draw layouts — no lighting headers:

```text
scene.vert
  ├── scene_ubo.glsl      → layout(set=0,binding=0) SceneUBO ubo
  └── scene_push.glsl     → layout(push_constant) Push push
```

Both files use include guards (`SPARK_SCENE_UBO_GLSL`, `SPARK_SCENE_PUSH_GLSL`) so double-inclusion from nested headers cannot redefine bindings.

### scene.frag — layered include stack

The fragment stage pulls shared layouts first, then feature modules:

```glsl
// shaders/scene.frag (lines 19–29)
#include "scene_ubo.glsl"
#include "scene_push.glsl"
#include "clustered_lights.glsl"
#include "color_space.glsl"
#include "ibl.glsl"
#include "gltf_map_uv.glsl"
```

Nested includes expand further — `clustered_lights.glsl` expects `ubo.clusterGrid` from the UBO header; `ibl.glsl` reads `ubo.iblParams` and samplers declared in the main file. Order matters: UBO and push must precede any file that references `ubo` or `push`.

### CMake DEPENDS mirrors the transitive graph

Spark lists shared headers on every custom command so any edit invalidates dependent SPIR-V:

```cmake
# CMakeLists.txt (line 436)
DEPENDS ${SHADER}
        ${SHADER_DIR}/clustered_lights.glsl
        ${SHADER_DIR}/equirect.glsl
        ${SHADER_DIR}/ibl.glsl
        ${SHADER_DIR}/punctual_shadows.glsl
        ${SHADER_DIR}/scene_push.glsl
        ${SHADER_DIR}/scene_ubo.glsl
        ${SHADER_DIR}/post_common.glsl
```

Note that `scene.frag` includes `punctual_shadows.glsl` transitively but CMake still lists it explicitly — Ninja cannot infer `#include` edges from GLSL text. When you add a new shared file (e.g. `gltf_pbr_extensions.glsl`), add it to **every** `add_custom_command` DEPENDS that includes it, or stale `.spv` files will ship.

### Compile-time paste, not runtime linking

All `#include` directives expand to one translation unit before `glslangValidator` emits SPIR-V. There is no separate link step for `.glsl` files — a typo in an include breaks every shader that pulls it. Manual compile helps isolate which file failed:

```bash
glslangValidator -V --target-env vulkan1.0 -Ishaders shaders/scene.frag -o /tmp/scene.frag.spv
```

## 2. Core Concepts

### Enabling includes in Vulkan GLSL

```glsl
#version 450
#extension GL_GOOGLE_include_directive : require

#include "scene_ubo.glsl"
```

`glslangValidator` resolves paths relative to `-I` directories. Spark passes `-Ishaders` in CMake:

```cmake
${GLSLANG_VALIDATOR} -V --target-env vulkan1.0 -I${SHADER_DIR} ${SHADER} -o ${SPV}
```

Without the extension and `-I`, `#include` fails at compile time with "cannot open include file."

### Include guards

Shared headers use ifndef guards:

```glsl
#ifndef SPARK_SCENE_UBO_GLSL
#define SPARK_SCENE_UBO_GLSL
layout(std140, set = 0, binding = 0) uniform SceneUBO { ... } ubo;
#endif
```

Double-including the same UBO in one translation unit would be a redefinition error. Guards make include order forgiving.

### Spark include graph (scene pass)

```text
scene.vert
  └── scene_ubo.glsl

scene.frag
  ├── scene_ubo.glsl
  ├── clustered_lights.glsl
  ├── color_space.glsl
  ├── ibl.glsl
  └── (punctual_shadows via clustered or frag — see repo)

shadow_depth.vert / .frag
  └── scene_ubo.glsl (viewProj, shadow matrices)

tonemap.frag
  └── post_common.glsl
```

| File | Responsibility |
|------|----------------|
| `scene_ubo.glsl` | Binding 0 frame UBO |
| `clustered_lights.glsl` | SSBO layouts, cluster iteration, attenuation |
| `color_space.glsl` | sRGB ↔ linear, tonemap helpers |
| `ibl.glsl` | Specular/diffuse environment sampling |
| `punctual_shadows.glsl` | Point/spot shadow atlas sampling |
| `post_common.glsl` | Shared post uniforms and push |
| `sprite_instance.glsl` | Per-sprite instance data layout |

### Sharing bindings across passes

When `shadow_depth.vert` includes `scene_ubo.glsl`, it **must** use the same `set` and `binding` numbers as `scene.vert` for descriptor set 0. The shadow pass descriptor layout only needs binding 0 (UBO), not binding 1 (textures) — but the UBO block definition must match byte-for-byte.

Mismatch symptom: shadow map depth wrong while color pass looks fine — `viewProj` or `worldToShadowClip` read at wrong offsets in shadow shader only if it used a different header copy.

### Functions vs inline code in includes

Spark puts **reusable functions** in includes:

```glsl
// color_space.glsl (conceptual)
vec3 sparkSrgbToLinear(vec3 c) { ... }
```

Main shader files keep **orchestration** — `main()` calls `evaluateClusteredLights()`, `sampleIbl()`, etc. Rule of thumb: if two `.frag` files need it, move to `.glsl` include.

### CMake DEPENDS and stale SPIR-V

`add_custom_command` must list every included file:

```cmake
DEPENDS ${SHADER} ${SHADER_DIR}/scene_ubo.glsl ${SHADER_DIR}/clustered_lights.glsl ...
```

Ninja uses DEPENDS for incremental builds. Omit `ibl.glsl` → edit IBL → `scene.frag.spv` timestamp older than header → runtime loads stale SPIR-V → hours lost debugging "IBL change did nothing."

### Include depth and circular dependencies

GLSL has no `#pragma once` standard — use guards. Avoid:

```text
a.glsl includes b.glsl
b.glsl includes a.glsl  // error
```

Spark keeps a **DAG**: leaf headers (`scene_ubo.glsl`) have no includes; mid-tier (`clustered_lights.glsl`) may include UBO; top-level `.frag` includes mid-tier.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Flat shaders/ directory

**Concept:** `-Ishaders` resolution

**Decision:** All includes live beside stage files, no nested `shaders/common/`

**Why:** Simple CMake DEPENDS globs; short include paths `"scene_ubo.glsl"`.

**In Spark:** Repository `shaders/` root.

### scene_ubo.glsl as single UBO source

**Concept:** Sharing bindings across passes

**Decision:** Never redeclare `SceneUBO` inline in a stage file

**Why:** One place to update 656-byte layout; all passes stay aligned.

**In Spark:** Included by scene, shadow, and related stages.

### Split lighting into clustered + ibl + shadows

**Concept:** Functions vs inline

**Decision:** Separate files by feature domain, not by shader stage

**Why:** IBL changes do not require editing cluster iteration; reviewers find code faster.

**In Spark:** `clustered_lights.glsl`, `ibl.glsl`, `punctual_shadows.glsl`.

### Explicit DEPENDS list over glob

**Concept:** CMake DEPENDS

**Decision:** Hand-list includes in CMake foreach loop

**Why:** `file(GLOB)` misses new files until reconfigure; explicit list fails review when adding includes.

**In Spark:** `CMakeLists.txt` shader custom command block.

### sprite_instance.glsl for 2D passes

**Concept:** Include graph

**Decision:** 2D sprite shaders include instance layout header separate from scene UBO

**Why:** Different pipeline layout — mixing layouts in one mega-header creates binding confusion.

**In Spark:** `sprite_instance.glsl` used by sprite vert/frag only.

## 4. Implementation Walkthrough

### Spark Files We Touch This Chapter

- `shaders/*.glsl` (all include headers)
- `shaders/scene.frag` (top of include chain)
- `CMakeLists.txt` (`SHADER_SOURCES`, DEPENDS)
- `shaders/shadow_depth.vert` (minimal include example)

### Step 1 — Extract a helper

Move a function from `scene.frag` to new `my_debug.glsl`:

```glsl
#ifndef SPARK_MY_DEBUG_GLSL
#define SPARK_MY_DEBUG_GLSL
vec3 heatmap(float t) { return vec3(t, 1.0 - t, 0.0); }
#endif
```

`#include "my_debug.glsl"` in `scene.frag`. Add file to CMake DEPENDS.

### Step 2 — Verify rebuild

Touch `my_debug.glsl` only. Run `ninja scene.frag.spv` — must rebuild. Touch unrelated file — must not.

### Step 3 — Shadow pass parity

Open `shadow_depth.vert`. Confirm it includes `scene_ubo.glsl` and uses `ubo.viewProj` — same symbol as scene pass.

### Step 4 — Fail intentionally

Remove include guard from `scene_ubo.glsl`, include twice in one file — read glslang redefinition error, restore guard.

### Step 5 — Document graph

Draw your engine's include DAG on paper. Mark which descriptors each header assumes (set/binding).

## 5. How This Connects to the Rest of Your Engine

Chapter **4** defined `scene_ubo.glsl` content. Part **V** extends `punctual_shadows.glsl`. Part **VIII** uses `post_common.glsl`. Every new pass should **include, not copy** shared layouts.

Good includes are how a 500-line `scene.frag` stays comprehensible without 500-line duplication across ten passes.

## 6. Pitfalls and Debugging

- **Include not found** — wrong `-I` path; run glslang manually with `-Ishaders`.
- **Stale SPIR-V** — missing CMake DEPENDS; `touch scene.frag` forces rebuild workaround.
- **Different UBO in two headers** — never fork `scene_ubo.glsl`; extend the original.
- **Macro leakage** — define `#define` in include without guard namespace; prefer prefixed names `SPARK_*`.
- **MoltenVK include paths** — spaces in path break some toolchains; keep shader dir simple.

## 7. Exercises

1. List all files that include `scene_ubo.glsl` (grep Spark repo).
2. Add a shared `constants.glsl` with `const float PI` — remove duplicate from `scene.frag`.
3. Remove one DEPENDS entry, change header, confirm stale behavior, then fix.
4. Create minimal `debug_flat.frag` that includes only `color_space.glsl` and outputs linear gray.
5. Compare compile time: monolithic `scene.frag` vs current include structure (expect similar; includes are compile-time only).

## Summary

Spark shares UBO layouts, lighting, color space, and IBL through guarded `.glsl` headers included from stage files. Enable `GL_GOOGLE_include_directive`, pass `-Ishaders` to glslang, and list every header in CMake DEPENDS. Treat the include graph as part of your API contract — same discipline as C++ headers.

---

Next: [Depth Testing and Face Culling](07-depth-culling.md).
