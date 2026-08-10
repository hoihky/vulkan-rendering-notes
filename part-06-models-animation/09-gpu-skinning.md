# GPU Skinning

> **Build-Along Milestone:** An animated skinned character deforms correctly in the forward pass and casts skinned shadows.

**You will learn:**

- Explain linear blend skinning (LBS) on the GPU with a joint palette SSBO.
- Trace `jointPalette` from `AnimatorComponent` through submit to `vkCmdDrawIndexed`.
- Relate packed joint vertex attributes to `SkinnedMesh` and `VulkanSceneVertexLayout`.
- Debug bind-pose vs animated silhouettes in RenderDoc.

## 1. The Problem We Are Solving

A skinned mesh has thousands of vertices, each influenced by up to four bones. Updating every vertex on the CPU each frame — multiplying bind-space positions by blended bone matrices — does not scale to hero characters at 60 FPS.

**GPU skinning** moves that work to the vertex shader: upload one matrix per joint once per draw, let the GPU blend per vertex in parallel. Spark uses **linear blend skinning** (LBS) in `scene.vert` with:

- Joint indices and weights in the vertex buffer (from `VulkanCustomMeshPool`)
- Joint matrices in SSBO binding 2 (`SkinSSBO`)
- `push.useSkinning` and `push.jointCount` to branch rigid vs skinned paths

This chapter connects Part VI animation (`jointPalette`) to Part III shader code (`scene.vert`) and the opaque pass upload in `VulkanSceneOpaquePass`.

## 2. Core Concepts

### Linear blend skinning math

For vertex position **p** in bind pose, bone indices **jᵢ** and weights **wᵢ** (Σwᵢ = 1):

```text
p' = Σ wᵢ · M[jᵢ] · p
```

| Symbol | Meaning |
|--------|---------|
| **p** | Position in mesh bind space |
| **M[j]** | Skin matrix for joint j (palette entry) |
| **p'** | Skinned position in bind space, before `push.model` |

Normals and tangents use the **linear 3×3 part** of blended bone matrices, then model inverse-transpose — approximation cheaper than full inverse-transpose per bone.

### Skin matrix on CPU

`Skeleton::BuildPaletteFromPose` fills:

```text
palette[j] = jointGlobalPrefix[j] * worldJoint[j] * inverseBind[j]
```

`SceneSubmit` calls `anim->ComputeJointPalette` → `SceneDrawItem.jointPalette`. This array is **not** in the custom mesh pool — it changes every frame while vertex attributes (joint indices/weights) stay static.

### Data flow per skinned draw

```text
AnimatorComponent::ComputeJointPalette
    → SceneDrawItem.jointPalette (CPU, per draw)
        → memcpy → Skin SSBO (GPU, binding 2, per draw)
            → scene.vert reads skin.bones[ji]
                → worldPos → ubo.viewProj → clip
```

Rigid draws: `useSkinning = 0`, SSBO unused, positions use `push.model` only.

### Single shader branch

Spark does **not** compile a separate skinned pipeline. One `scene.vert` / `scene.frag` pair serves rigid props and characters:

```glsl
if (push.useSkinning != 0) {
    // 4-bone accumulation loop
} else {
    // standard model * attribute
}
```

`VulkanCustomMeshPool` always packs joint/weight slots — rigid meshes get zero weights on bone 0.

### Skin SSBO layout

```glsl
layout(std430, set = 0, binding = 2) readonly buffer SkinSSBO {
    mat4 bones[];
} skin;
```

C++ allocation:

```cpp
static constexpr std::uint32_t kMaxSkinJoints = 64;
static constexpr VkDeviceSize kSkinSsboBytes = 4096;  // 64 × 64 bytes
```

One mapped SSBO per frame-in-flight; **each skinned draw** overwrites the buffer before `vkCmdDrawIndexed` (not persistent across draws).

### Packed joint indices (MoltenVK)

```glsl
layout(location = 4) in vec4 inJointsPacked;  // float bits → uint
uvec4 inJoints = uvec4(
    floatBitsToUint(inJointsPacked.x), ...);
```

Integer vertex attributes caused fetch issues on some Apple/MoltenVK paths — Spark stores indices as float bit patterns in the interleaved buffer.

### Shadow pass parity

`shadow_depth.vert` includes the same `SkinSSBO` and skinning loop so cascaded shadows match the deformed silhouette. Skipping GPU skinning in shadow passes produces **bind-pose shadows** — obvious on walking characters.

### jointCount guard

Shader skips influences where `ji >= push.jointCount`. CPU must set `jointCount` to palette size and upload at least that many matrices — mismatch freezes limbs or causes garbage deformation.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Four influences per vertex

**Concept:** Linear blend skinning

**Decision:** Fixed loop `k = 0..3` in vertex shader

**Why:** Matches glTF `JOINTS_0` / `WEIGHTS_0` vec4; industry standard for real-time.

**In Spark:** `scene.vert` skinning loop.

### Per-draw SSBO memcpy

**Concept:** Data flow

**Decision:** Overwrite skin SSBO each skinned draw, not per-frame once

**Why:** Multiple skinned characters may share a frame slot sequentially; last draw wins without per-draw offset (simple, sufficient for demo scale).

**In Spark:** `VulkanSceneOpaquePass.cpp` lines 127–138.

### No dual quaternion skinning

**Concept:** LBS limitations

**Decision:** Matrix LBS only

**Why:** Simpler shader; candy-wrapper artifacts at extreme weights acceptable in teaching scope.

**In Spark:** Comment in design — future quality upgrade path.

### Shared 20-float vertex layout

**Concept:** Packed joint indices

**Decision:** Rigid and skinned share `VulkanSceneVertexLayout`

**Why:** One custom pool, one pipeline vertex input state.

**In Spark:** `VulkanSceneVertexLayout.hpp` — 20 floats stride.

### useSkinning in push constants

**Concept:** Single shader branch

**Decision:** Branch in VS, not specialization constant

**Why:** Per-draw toggle without extra pipeline variants.

**In Spark:** `ModelPushConstants::useSkinning` — 192-byte push block.

## 4. Implementation Walkthrough

### Spark Files We Touch This Chapter

- `shaders/scene.vert`, `shaders/shadow_depth.vert`
- `src/spark/render/scene/VulkanSceneOpaquePass.cpp`
- `include/spark/render/scene/VulkanSceneDescriptors.hpp`
- `src/spark/scene/SceneSubmit.cpp` (joint palette fill)
- `include/spark/scene/SkinnedMesh.hpp`

### Step 1 — Verify vertex inputs

Confirm locations 0–5 match pool packer output. RenderDoc mesh viewer on skinned draw — joint attributes non-zero on limbs.

### Step 2 — Read skinning loop

Study accumulation in `scene.vert` — note `continue` on `w <= 0` and `ji >= jointCount`.

### Step 3 — Trace submit palette

Breakpoint in `SkinnedSubmitSink` when `ComputeJointPalette` runs — compare matrix at root joint vs finger joint during walk clip.

### Step 4 — Trace GPU upload

In opaque pass record, verify `memcpy` size = `jc * sizeof(Matrix4)` before push and draw.

### Step 5 — Toggle useSkinning

Force `push.useSkinning = 0` in C++ for skinned draw — mesh snaps to bind pose; confirms branch and attributes wired.

### Step 6 — Shadow capture

RenderDoc shadow pass draw — confirm skinning enabled on `shadow_depth.vert` stage.

### Step 7 — Milestone

Play walk animation under directional sun (Part IV) — lighting and normals follow deformation; no bind-pose shadow on ground.

## 5. How This Connects to the Rest of Your Engine

**Custom Mesh Pool** (Chapter 5) packs static joint indices/weights. **Animator** (Chapter 8) produces palette. **Part IV** lighting uses deformed `vWorldPos` / `vNormal`. **Part V** camera unchanged. **Part VII+** materials sample textures on deformed UVs.

GPU skinning is the **last mile** of the animation pipeline before rasterization.

## 6. Pitfalls and Debugging

- **Exploded mesh** — wrong `inverseBind` or palette formula; check bind pose in isolation.
- **Frozen limb** — weight on joint index ≥ `jointCount` or zero palette row.
- **Jitter at seams** — weights not normalized to 1.0 on import.
- **Rigid shadow, skinned color** — shadow shader missing skinning branch.
- **SSBO stale between draws** — two characters, second overwrites first if draws batched wrong — verify draw order and memcpy per draw.

## 7. Exercises

1. Visualize `inWeights` sum as grayscale — should be 1.0 everywhere on clean glTF.
2. Output `jointCount` as debug color in fragment shader for one draw.
3. Compare `sizeof(Matrix4)` to std430 `mat4` stride in SSBO — confirm 64 bytes.
4. Animate only root joint in palette — entire mesh should rigidly follow.
5. List maximum skinned draws per frame before SSBO overwrite becomes a bug in Spark's current per-draw upload model.

## Summary

GPU skinning in Spark blends up to four bone influences per vertex in `scene.vert`, reading joint matrices from SSBO binding 2 uploaded per draw from `SceneDrawItem.jointPalette`. Push constants select the skinning branch and guard joint indices. Shadow passes use the same path for correct silhouettes. Master the CPU palette formula before debugging shader deformation.

---

Next: [Locomotion FSM](10-locomotion-fsm.md).
