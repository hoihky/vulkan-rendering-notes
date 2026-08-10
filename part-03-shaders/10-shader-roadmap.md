# Shader Roadmap and Next Steps

> **Build-Along Milestone:** You can inventory Spark's shader passes, bindings, and plan the next features (textures, shadows, post) without shader architecture debt.

**You will learn:**

- Map Spark shader files to render passes and descriptor bindings.
- Choose between shader variants, dynamic branches, and separate pipelines.
- Plan the path from Part III forward shading to advanced topics in later parts.

## 1. The Problem We Are Solving

Part III taught toolchain, vertex/fragment stages, UBOs, push constants, includes, depth, blend, and debugging. Your engine now draws lit meshes with transparency and sky — but Spark ships **many more passes**: shadows, particles, sprites, UI, tonemap, post.

Before diving into Part IV (textures and lighting depth), you need a **roadmap**: what exists, what binds where, and how to extend without combinatorial explosion. This chapter is the capstone inventory and forward plan.

## 2. Core Concepts

### Spark shader pass inventory

| Pass / stage files | Purpose | Key bindings |
|--------------------|---------|--------------|
| `scene.vert` / `scene.frag` | Forward HDR PBR scene | UBO 0, tex array 1, skin SSBO 2, lights SSBO 3+ |
| `shadow_depth.vert` / `.frag` | Directional/punctual depth | UBO 0 |
| `particles.vert` / `.frag` | GPU particles | Pass-specific set |
| `sprite.vert` / `.frag` + `sprite_instance.glsl` | 2D sprites | Instance push, atlas |
| `text.vert` / `.frag` | SDF text | Font atlas |
| `ui.vert` / `.frag` | Screen UI quads | UI atlas |
| `tonemap.frag` | HDR → display | HDR texture, exposure UBO |
| `post_process.frag` | Bloom, color grade | Ping-pong HDR |

Each pass owns a `VkPipelineLayout` — **do not assume** set 0 means the same thing across passes. Scene set 0 binding 1 is `sceneTextures`; UI set 0 may be an entirely different layout.

### Descriptor binding table (scene set 0)

Document and maintain a table like this in your engine wiki:

| Binding | Type | Stages | Content |
|---------|------|--------|---------|
| 0 | Uniform buffer | VS, FS | `SceneUBO` (656 B) |
| 1 | Combined image sampler | FS | `sampler2DArray sceneTextures` |
| 2 | Storage buffer | VS | Skin bone palette |
| 3+ | Storage buffers | FS | Cluster lights, shadow atlas data |

Spark centralizes scene layout creation in descriptor setup code (see `VulkanSceneDescriptors` in Spark repo). When adding binding 4, update **layout, pool, writes, and all shaders** — validation catches misses, but only at first draw.

### Three ways to specialize shaders

| Technique | When to use | Spark example |
|-----------|-------------|---------------|
| **Separate pipelines** | Fixed state differs (blend, depth) | lit / sky / transparent |
| **Runtime branch** | Few modes, same pipeline | `shadingModel`, `skyMode` |
| **Compile-time variants** | Many permutations, hot paths | Optional future normal-map-only perm |

Avoid `#define` explosion:

```text
10 optional features → 2^10 = 1024 variants if combined blindly
```

Spark prefers runtime branches for toon/PBR/unlit (3 modes) and separate pipelines for blend/depth — a pragmatic middle ground.

### Shader variants build system (future)

If you add compile-time variants:

```cmake
# Conceptual — not in Spark baseline
foreach (DEF NORMAL_MAP ON OFF)
  glslang -DUSE_NORMAL_MAP=${DEF} ...
endforeach()
```

Curate **permutations** — only ship combinations you test. Cache `.spv` in build dir with hash names (`scene.frag.spv`, `scene_nm.frag.spv`).

### Compute shader entry points (later parts)

Vulkan compute enables:

- Frustum/cluster light culling
- SSAO, blur passes
- Texture mip generation

Compute shares **descriptor sets** with graphics but uses `VkPipelineLayout` with `VK_SHADER_STAGE_COMPUTE_BIT`. Spark's clustered lighting may use CPU build today; moving cluster assignment to compute is a natural Part IX+ optimization.

### Bindless and descriptor indexing (advanced)

When draw counts reach thousands and `textureLayer` push is insufficient:

- `GL_EXT_descriptor_indexing` — non-uniform `sampler2D` array index
- Bindless texture heap — single large descriptor array

Spark stays with **texture2DArray + layer index** for portability. Document when your project outgrows it: typically open-world with hundreds of unique materials per frame.

### Forward vs deferred decision point

Part I previewed forward vs deferred. After Part III you have forward PBR working. Move to deferred when:

- Dynamic light count exceeds clustered forward budget
- Material decoupling from light count is worth G-buffer complexity

Spark commits to **clustered forward** through Part VII — you should too until profiling proves otherwise.

### Roadmap aligned to book parts

| Part | Shader focus |
|------|----------------|
| IV | Texture formats, samplers, material sampling in `scene.frag` |
| V | Shadow map shaders, `punctual_shadows.glsl` depth |
| VI | Skinning already in `scene.vert`; animation drives SSBO |
| VII | Particles — additive blend, different vertex inputs |
| VIII | Tonemap, exposure, `post_common.glsl` |
| IX–XVI | Specialized passes, optimization, bindless optional |

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Texture array over bindless (for now)

**Concept:** Bindless advanced

**Decision:** `sampler2DArray` + `textureLayer` push

**Why:** Vulkan 1.0 wide support; simpler descriptor model for book scope.

**In Spark:** `scene.frag` binding 1.

### Includes over monolith

**Concept:** Pass inventory

**Decision:** Shared `.glsl` headers per domain

**Why:** Maintainability as `scene.frag` grows past 500 lines.

**In Spark:** Chapter 6 include graph.

### Runtime shadingModel branch

**Concept:** Three specialization ways

**Decision:** Integer branch for PBR/toon/unlit

**Why:** Three modes do not justify 3× shader compile matrix.

**In Spark:** `push.shadingModel` in `scene.frag`.

### Separate post shader chain

**Concept:** Pass inventory

**Decision:** `tonemap.frag` / `post_process.frag` off main scene SPIR-V

**Why:** Different render pass attachments; no risk of scene UBO mismatch in post.

**In Spark:** Post passes in `src/spark/render/post/`.

### CMake SparkShaders aggregate

**Concept:** Variants build system

**Decision:** One custom target builds all SPIR-V

**Why:** Engine link depends on all shaders; CI fails on any compile error.

**In Spark:** Chapter 1.

## 4. Implementation Walkthrough

### Step 1 — Build binding spreadsheet

From Spark source, list every `layout(set =` in `shaders/`. One row per binding per pass.

### Step 2 — Draw pass diagram

```text
Shadow depth passes → Scene HDR (opaque → transparent) → Tonemap → UI → Present
```

Mark which shader files attach to each arrow.

### Step 3 — Identify your next gap

If following the book linearly, Part IV is texture sampling depth — verify `sceneTextures` descriptor write and mip chain before shadow math.

### Step 4 — Variant policy document

Write one page: when you will add a new pipeline vs a new `#ifdef` variant vs a branch. Stick to it for three features.

### Step 5 — Tech debt audit

List temporary debug `outColor` hacks, duplicated `Push` blocks, and missing CMake DEPENDS — fix before Part IV.

## 5. How This Connects to the Rest of Your Engine

Part III is the **shader foundation**. Parts IV–VIII add data and passes on top without rewriting the vertex transform or UBO frame contract. Parts IX+ optimize and specialize.

Revisit this chapter when adding a new pass — update inventory table before writing GLSL.

## 6. Pitfalls and Debugging

- **Copy scene descriptor layout to UI** — binding numbers collide at runtime.
- **Variant combinatorics** — team cannot test all permutations; ship curated set only.
- **Premature bindless** — months of descriptor complexity before profiling proves need.
- **Deferred too early** — lose momentum; forward clustered carries far.
- **No binding doc** — every new programmer re-derives from RenderDoc.

## 7. Exercises

1. Complete binding table for scene set 0 from Spark headers — verify against RenderDoc one capture.
2. Count lines of GLSL per pass — which pass deserves include split next?
3. Propose one compile-time variant Spark might add (e.g. `NO_IBL`) — how many permutations?
4. Sketch compute pass for cluster assignment — inputs, outputs, barrier with graphics.
5. Write a one-paragraph "forward enough for our game" criteria for a hypothetical project.

## Summary

Part III equipped you to author, compile, bind, and debug Spark-style forward shaders. Spark's pass inventory spans scene, shadow, 2D, UI, and post — each with its own pipeline layout. Extend via includes and curated variants; add bindless or deferred only when profiling and scope demand. Continue to [Part IV — Textures and Lighting](../part-04-textures-lighting/index.md) for material and light sampling depth.

---

Continue to [Part IV — Textures and Lighting](../part-04-textures-lighting/index.md).
