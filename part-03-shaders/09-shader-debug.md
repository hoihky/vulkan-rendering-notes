# Debugging Shaders

> **Build-Along Milestone:** You have a repeatable workflow: glslang for compile errors, validation layers for pipeline/descriptor mismatches, RenderDoc for pixel truth.

**You will learn:**

- Triangulate shader bugs across compile time, record time, and GPU execution.
- Use validation layers and RenderDoc on Spark's scene pass.
- Apply shader debug visualization techniques without permanent code clutter.

## 1. The Problem We Are Solving

Shader bugs are painful because the GPU is a black box. Symptoms are vague: "black screen," "wrong colors," "flickering." The failure might be:

1. **Compile time** — GLSL syntax or link mismatch between stages
2. **Pipeline creation** — SPIR-V does not match vertex layout or push range
3. **Record time** — wrong descriptor, push constants not updated
4. **Execution** — math error, wrong space, gamma mistake

Professional debugging uses **layered tools** — each layer falsifies hypotheses. This chapter builds a checklist around Spark's `scene.vert` / `scene.frag` so you spend minutes, not afternoons, on each class of bug.

### The four layers of shader debugging (beginner)

When the screen is wrong, work **top to bottom** — never jump to rewriting BRDF math first.

| Layer | Tool | Question answered |
|-------|------|-------------------|
| **1. Compile** | `glslangValidator`, CMake build | Does GLSL compile to SPIR-V? |
| **2. Pipeline** | Vulkan validation layers | Does pipeline match vertex layout, render pass, push size? |
| **3. Bindings** | Validation + RenderDoc | Are descriptors, push constants, buffers bound correctly? |
| **4. Math** | RenderDoc pixel history, debug colors | Is the shader logic wrong? |

```text
Black screen workflow:
  1. Did CMake build .spv? (layer 1)
  2. Any validation error on vkCmdDraw? (layer 2–3)
  3. RenderDoc: is draw call present? Pipeline bound? (layer 3)
  4. Replace outColor = vec4(1,0,0,1) in frag (layer 4)
```

**Debug visualization trick:** output a solid color or `vTexCoord` as RGB temporarily. If solid red appears, pipeline and bindings work — bug is in lighting math. If still black, problem is earlier.

**RenderDoc for beginners:** capture one frame, find `vkCmdDrawIndexed` for your mesh, inspect bound vertex buffer, push constants, and fragment shader output at a pixel.

## 1b. Concept Deep Dive — Debugging scene.vert / scene.frag End-to-End

### Layer 1 — compile the exact translation unit

Scene shaders pull multiple includes — always compile with the same flags CMake uses:

```bash
glslangValidator -V --target-env vulkan1.0 -Ishaders shaders/scene.vert -o /tmp/scene.vert.spv
glslangValidator -V --target-env vulkan1.0 -Ishaders shaders/scene.frag -o /tmp/scene.frag.spv
```

Common failures: missing `-Ishaders` (cannot open `scene_ubo.glsl`), undefined `ubo` (include order wrong), duplicate binding (forgot include guard). Fix at compile time before launching SparkDemo.

### Layer 2 — validate layout contracts

Three size proofs must pass before draw debugging:

| Contract | Proof | Symptom if wrong |
|----------|-------|------------------|
| UBO | `sizeof(SceneUniformGpu) == 656` | Sheared shadows, wrong sun color |
| Push | `sizeof(ModelPushConstants) == 348` | Scrambled transforms / map layers |
| Vertex | 80-byte stride, locations 0–7 | Exploded mesh, swimming UVs |

Validation layers flag push size mismatches (`VUID-vkCmdPushConstants-*`) and vertex location gaps (`VUID-VkGraphicsPipelineCreateInfo-pVertexInput-*`).

### Layer 3 — RenderDoc scene-pass checklist

For one `vkCmdDrawIndexed` in the opaque pass, verify in order:

1. **Pipeline** — `pipelineLit` vs `pipelineSky` vs `pipelineLitTransparent`
2. **Descriptor set 0** — UBO binding 0, `sceneTextures` binding 1, skin SSBO binding 2 (skinned only)
3. **Push constants** — first 64 bytes = `model` matrix; `textureLayer` at offset 68; `shadingModel` at offset 104
4. **Vertex inputs** — six or eight attributes depending on mesh type

### Layer 4 — surgical debug outputs in scene.frag

Replace `main()` body temporarily — rebuild SPIR-V after each change:

```glsl
// Normal visualization
outColor = vec4(normalize(vNormal) * 0.5 + 0.5, 1.0);

// UV visualization
outColor = vec4(vTexCoord0, 0.0, 1.0);

// Push constant reach test
if (push.shadingModel == 99) outColor = vec4(1.0, 0.0, 1.0, 1.0);
```

If solid magenta appears but PBR is black, bindings and pipeline are fine — isolate lighting by commenting out `#include "clustered_lights.glsl"` and testing directional-only sun from `ubo.lightDir`.

### Stale SPIR-V — the silent layer zero

If debug color changes never appear despite successful compile, compare `.spv` timestamps in `${CMAKE_BINARY_DIR}/shaders/` against your edited `.glsl`. Missing CMake DEPENDS on an include file is the most common cause — touch `scene.frag` or run `cmake --build . --target SparkShaders` clean.

### Step-by-step RenderDoc capture guide (Spark)

**RenderDoc** is a free GPU debugger. It records one frame of Vulkan commands and lets you inspect every draw call, bound buffer, shader, and pixel output — like a time machine for the GPU.

**What a "capture guide" means:** numbered steps to go from "something looks wrong" to "I can see exactly which `vkCmdDrawIndexed` failed and what data the shader received."

#### One-time setup (macOS / Linux)

1. Install [RenderDoc](https://renderdoc.org/) (1.29+ supports Vulkan on Apple Silicon via MoltenVK).
2. Build Spark in **Debug** or **RelWithDebInfo** so validation layers and symbols are available.
3. Launch RenderDoc → **File → Launch Application**.
4. **Executable:** path to your demo binary (e.g. `build/bin/ThreeDDemo`).
5. **Working directory:** Spark repo root (so relative asset paths resolve).
6. **Environment:** optional `VK_INSTANCE_LAYERS=VK_LAYER_KHRONOS_validation` if not enabled in-engine.

#### Capture one frame

1. Click **Launch** — demo window opens inside RenderDoc's hook.
2. Reproduce the bug (black mesh, wrong texture, etc.).
3. Press **F12** (default) or click **Capture Frame** in the RenderDoc overlay — saves a `.rdc` file.
4. The capture opens automatically in the Event Browser.

#### Navigate to your draw call

```text
Event Browser (left)
  └── Queue 0
        └── vkQueueSubmit
              └── Command Buffer N
                    ├── vkCmdBeginRenderPass  (HDR scene pass)
                    ├── vkCmdBindPipeline     (pipelineLit)
                    ├── vkCmdBindDescriptorSets (set 0)
                    ├── vkCmdPushConstants
                    └── vkCmdDrawIndexed      ← select this
```

**Tips for Spark:**

- Filter events by typing `DrawIndexed` in the search box.
- Sky draws use `pipelineSky`; opaque meshes use `pipelineLit`; particles use a separate pipeline.
- Multiple draws per frame — pick the draw whose **Pipeline** name matches the pass you care about.

#### Inspect pipeline state (Layer 3)

With a `vkCmdDrawIndexed` selected, open the **Pipeline State** tab:

| Tab | What to verify for Spark scene pass |
|-----|-------------------------------------|
| **Vertex Input** | Locations 0–2 (rigid) or 0–7 (skinned); stride 80 bytes for custom pool |
| **Shader Stages** | `scene.vert` + `scene.frag` SPIR-V; click **Edit** to see disassembly |
| **Descriptor Sets** | Set 0: UBO (binding 0), `sceneTextures` (1), Skin SSBO (2 if skinned) |
| **Push Constants** | `model` matrix at offset 0; `textureLayer` at byte 68; `shadingModel` at 104 |

Red highlights mean bound resource is null or incompatible — common when descriptor not updated.

#### Inspect mesh data

1. **Mesh Output** tab → click a vertex on the preview.
2. Check **VS Input** attributes: position, normal, UV, joint indices (skinned).
3. **VS Output** / **PS Input**: interpolated varyings reaching the fragment shader.

If VS Input positions look correct but VS Output `gl_Position` is off-screen, suspect `ubo.viewProj` or `push.model`.

#### Pixel debugging (Layer 4)

1. **Texture Viewer** tab → click a pixel on the mesh.
2. **History** shows which draw wrote that pixel (useful when something is occluded).
3. **Debug** picker → choose **PS Output** to see final fragment color before blend.
4. Enable **Highlight Drawcall** to see only geometry from the selected draw.

**Workflow for "black mesh":**

```text
1. Is there a DrawIndexed for your mesh?  → No: submit/cull bug (CPU)
2. Pipeline bound?                        → No: record order bug
3. Push model matrix sane?                → No: SceneSubmit transform
4. textureLayer = -1 but expecting map?   → Material submit / upload
5. PS output black but VS OK?             → lighting math — try debug color in frag
```

#### Export and share

**File → Export** saves `.rdc` for sharing with teammates. Captures include shaders and buffers — no need to reproduce locally if you attach the file to a bug report.

## 2. Core Concepts

### Layer 1: Offline compile (glslangValidator)

Catch syntax and type errors before running the app:

```bash
glslangValidator -V --target-env vulkan1.0 -Ishaders shaders/scene.frag -o /tmp/scene.frag.spv
```

| Error class | Example message | Fix |
|-------------|-----------------|-----|
| Syntax | `error: syntax error` | Line number in GLSL |
| Include | `cannot find include file` | Add `-Ishaders` |
| Link (pipeline create) | Varying mismatch | Match vert out / frag in |
| Binding | Duplicate `binding` | Include guard or merge layouts |

CMake runs this automatically — **read build output** when `SparkShaders` fails.

### Layer 2: Validation layers

Enable in Spark dev builds (`VK_LAYER_KHRONOS_validation`). Categories relevant to shaders:

| Message theme | Often means |
|---------------|-------------|
| `VUID-VkGraphicsPipelineCreateInfo-*` | Vertex input vs shader location |
| `VUID-vkCmdPushConstants-*` | Size/stage mismatch |
| `VUID-vkCmdDraw-*` | Bound pipeline/layout/descriptor wrong |
| `UNASSIGNED-CoreValidation-Shader-*` | Interface not consumed |

Validation catches **record-time** mistakes. It does not catch `normalize(zeroVector)` or wrong light direction sign.

### Layer 3: RenderDoc capture

[RenderDoc](https://renderdoc.org/) captures one frame: every draw, pipeline state, descriptors, push constants, mesh inputs.

Workflow:

1. Launch `SparkDemo` through RenderDoc
2. Capture frame (F12 or trigger)
3. Event browser → select `vkCmdDrawIndexed` for scene mesh
4. Inspect:
   - **Pipeline State** → shader disassembly, push constants hex
   - **Mesh Viewer** → vertex inputs after VS
   - **Texture Viewer** → bound `sceneTextures` array
   - **Pixel History** → why depth test failed

RenderDoc is the ground truth when the image disagrees with your mental model.

### Layer 4: Shader debug output modes

Temporary fragment shader outputs diagnose without printf (GLSL `debugPrintfEXT` exists but needs setup):

| Debug mode | Code pattern | Reveals |
|------------|--------------|---------|
| Normal visualize | `outColor = vec4(N*0.5+0.5, 1)` | Normal direction, handedness |
| UV checker | `fract(vTexCoord*10)` | UV seams, orientation |
| Index heat | `outColor = vec4(push.textureLayer/15.0, ...)` | Wrong material layer |
| Light NdotL | `outColor = vec4(vec3(diff), 1)` | Light dir sign, shadow over-dark |
| Solid magenta | `outColor = vec4(1,0,1,1)` | Draw reached GPU at all |

Use push `shadingModel` or a UBO debug flag to switch modes without recompiling ten variants — or use C preprocessor `#define DEBUG_VIEW 1` for local sessions.

### Layer 5: Pipeline state parity checklist

When a draw looks wrong, verify in order:

```text
□ Correct VkPipeline (lit / sky / transparent)
□ VkPipelineLayout matches descriptor set layout
□ Descriptor set 0 bound with current frame's UBO + textures
□ Push constants size 192, updated this draw
□ Vertex buffer bound, index count > 0
□ Viewport/scissor match render area
□ Render pass begun on correct framebuffer (HDR vs swapchain)
```

Spark's multi-pass renderer makes **wrong pass** a common mistake — tonemap pass bound but scene shader expected.

### Common Spark-specific failure modes

| Symptom | Likely cause |
|---------|--------------|
| All black HDR | Emissive zero, light intensity zero, or tonemap exposure |
| Pink/magenta placeholder | Missing texture descriptor |
| Mesh exploded | Skinning SSBO garbage or wrong joint indices |
| Sky swims | Using `vWorldPos` for equirect instead of ray reconstruct |
| Shadows acne | `shadowParams` bias; debug with NdotL visualization |
| Wrong material | `textureLayer` push not updated between draws |

### SPIR-V inspection tools

When glslang succeeds but behavior is odd:

- RenderDoc → compile stats on shader
- `spirv-dis` / SPIRV-Cross disassemble `.spv` to check optimizations did not eliminate your branch

Rare on desktop; useful on mobile drivers with aggressive optimization.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Offline SPIR-V only

**Concept:** Layer 1 compile

**Decision:** No runtime GLSL compile in engine

**Why:** Errors surface in CI/build; identical SPIR-V in capture and shipping.

**In Spark:** Chapter 1 CMake pipeline.

### Validation in dev presets

**Concept:** Layer 2 validation

**Decision:** Enable Khronos validation in debug CMake preset

**Why:** Catches descriptor and push mistakes early; unacceptable overhead in release.

**In Spark:** Instance creation flags in device bootstrap (Part I).

### No printf in shipping shaders

**Concept:** Debug output modes

**Decision:** Color false-color modes over `GL_EXT_debug_printf`

**Why:** printf needs extension, buffer setup, and sync; false-color works everywhere.

**In Spark:** Ad-hoc `outColor` assignments during dev.

### static_assert on CPU-GPU layouts

**Concept:** Pipeline parity checklist

**Decision:** UBO and push offsetof checks in C++

**Why:** Validation does not compare C++ struct to GLSL; asserts catch before GPU.

**In Spark:** `VulkanSceneUniformWriter.cpp`, `VulkanSceneOpaquePass.cpp`.

### RenderDoc-friendly single draw

**Concept:** RenderDoc workflow

**Decision:** Scene demo can isolate one mesh draw via scene config

**Why:** Thousand-draw captures are hard to navigate; isolate hypothesis.

**In Spark:** Reduce `SceneDrawItem` list in test scenes.

## 4. Implementation Walkthrough

### Spark Files We Touch This Chapter

- Build: `CMakeLists.txt`, `glslangValidator`
- Runtime: validation-enabled instance
- Shaders: `scene.vert`, `scene.frag` (temporary debug edits)
- Tools: RenderDoc, optional `spirv-dis`

### Step 1 — Break compile intentionally

Add `syntax error` to `scene.frag`. Build `SparkShaders`. Read glslang line number. Revert.

### Step 2 — Break link intentionally

Change `vNormal` to `vec4` in fragment only. Run app — pipeline create fails. Read validation message for location mismatch.

### Step 3 — Capture baseline frame

RenderDoc capture SparkDemo with default scene. Bookmark one opaque draw event ID.

### Step 4 — Push constant diff

Change `metallic` on one material in scene JSON/code. Re-capture. RenderDoc push constant hex at offset 88 should change.

### Step 5 — False-color normal pass

```glsl
outColor = vec4(normalize(vNormal) * 0.5 + 0.5, 1.0);
```

Capture — compare to expected smooth sphere gradient.

### Step 6 — Document your checklist

Save a personal markdown checklist from Section 2 Layer 5 — paste into future bug tickets.

## 5. How This Connects to the Rest of Your Engine

Every Part III chapter assumes you can **verify** claims with tools. Part **IV** texture bugs need RenderDoc texture viewer. Part **V** shadows need pixel history depth compare. Part **XVI** performance tuning needs GPU timestamps after correctness is proven.

Debugging is a skill parallel to writing shaders — budget time to learn RenderDoc shortcuts early.

## 6. Pitfalls and Debugging

- **Debugging release build** — validation off, optimized out variables; use debug preset.
- **Stale SPIR-V** — shader change not rebuilt; always check `.spv` timestamp.
- **Capturing wrong process** — multiple Vulkan apps; confirm PID in RenderDoc.
- **Chasing math before parity** — always confirm descriptors/push before rewriting BRDF.
- **Leaving debug `outColor` in** — breaks lighting tests; use `#ifdef` or git stash discipline.

## 7. Exercises

1. Capture frame, export pipeline state JSON for one draw — list all blend/depth settings.
2. Find a validation warning in Spark startup log — explain in plain English.
3. Use Pixel History to see why a fragment failed depth test behind a wall.
4. Create three debug modes toggled by keyboard updating a UBO `debugMode` int.
5. Disassemble `scene.frag.spv` — locate `texture` instruction for array layer.

## Summary

Debug shaders in order: glslang compile, validation at pipeline/record, RenderDoc at execution. Use false-color fragment outputs to isolate varyings, normals, and material IDs. Keep a pipeline-descriptor-push checklist for every draw bug. Spark's offline SPIR-V and static_assert layout guards front-load failures to build time when possible.

---

Next: [Shader Roadmap and Next Steps](10-shader-roadmap.md).
