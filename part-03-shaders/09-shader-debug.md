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
