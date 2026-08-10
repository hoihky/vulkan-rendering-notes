# Particle Shaders

> **Build-Along Milestone:** Particles render as soft round discs facing the camera.

**You will learn:**

- Explain billboard corner reconstruction in particle.vert.
- Describe soft-disc alpha in particle.frag without a texture.
- Match ParticleUniformGpu C++ layout to GLSL ParticleUBO std140.

## 1. The Problem We Are Solving

Each particle is a **camera-facing quad**, but you want round fire and smoke — not cardboard squares. The shaders
must reconstruct world positions from a center point and corner offset, then fade alpha radially.

Build along: read particle.vert and particle.frag alongside the Record vertex pack loop.

## 2. Core Concepts

### Billboard vertex math

Vertex shader places each corner at `center + right * corner.x * size * 0.5 + up * corner.y * size * 0.5` using normalized camera axes from the UBO.


### Soft disc fragment

Fragment shader treats corner UV as a unit disc, smoothsteps alpha from center to edge, and discards fragments below threshold — round particles without a texture.


### UBO layout std140

ParticleUBO: mat4 viewProj, vec4 cameraPos, vec4 cameraRight, vec4 cameraUp — binding 0, vertex stage only.


### No texture sampling

Spark particles are procedural colored discs — color comes from interpolated per-vertex inColor.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Expand in VS not geometry shader

**Concept:** Billboard vertex math

**Decision:** Expand in VS not geometry shader

**Why:** Geometry shaders are absent on some mobile paths; explicit triangles are portable.

**In Spark:** `shaders/particle.vert` main

### smoothstep + discard

**Concept:** Soft disc fragment

**Decision:** smoothstep + discard

**Why:** Square quads become round glow without alpha texture — one less sampler in the pass.

**In Spark:** `shaders/particle.frag` edge fade

### Single uniform buffer binding

**Concept:** UBO layout std140

**Decision:** Single uniform buffer binding

**Why:** viewProj and camera basis updated once per frame for all particles.

**In Spark:** `ParticleUniformGpu` matches GLSL block

### Vertex color only

**Concept:** No texture sampling

**Decision:** Vertex color only

**Why:** Presets tune start/end Vector4; GPU multiplies alpha by radial edge.

**In Spark:** `SceneParticleInstance::color`

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `shaders/particle.vert`
- `shaders/particle.frag`
- `src/spark/render/sprites2d/VulkanParticlePass.cpp`

### Step 1 — Read vertex shader inputs and UBO

*Attributes, ParticleUBO, billboard world position.*

```glsl
// shaders/particle.vert (lines 1–25)
#version 450

layout(location = 0) in vec3 inPos;
layout(location = 1) in vec4 inColor;
layout(location = 2) in float inSize;
layout(location = 3) in vec2 inCorner;

layout(std140, set = 0, binding = 0) uniform ParticleUBO {
    mat4 viewProj;
    vec4 cameraPos;
    vec4 cameraRight;
    vec4 cameraUp;
};

layout(location = 0) out vec4 vColor;
layout(location = 1) out vec2 vDisc;

void main() {
    vec3 right = normalize(cameraRight.xyz);
    vec3 up = normalize(cameraUp.xyz);
    vec3 worldPos = inPos + right * inCorner.x * inSize * 0.5 + up * inCorner.y * inSize * 0.5;
    gl_Position = viewProj * vec4(worldPos, 1.0);
    vColor = inColor;
    vDisc = inCorner;
}
```

### Step 2 — Read fragment soft disc

*Radial edge fade and discard for square quad corners.*

```glsl
// shaders/particle.frag (lines 1–15)
#version 450

layout(location = 0) in vec4 vColor;
layout(location = 1) in vec2 vDisc;

layout(location = 0) out vec4 outColor;

void main() {
    float d = length(vDisc);
    float edge = 1.0 - smoothstep(0.82, 1.02, d);
    if (edge < 0.004) {
        discard;
    }
    outColor = vec4(vColor.rgb, vColor.a * edge);
}
```

### Step 3 — Match C++ UBO pack

*Memcpy viewProj and camera basis into ParticleUniformGpu.*

```cpp
// src/spark/render/sprites2d/VulkanParticlePass.cpp (lines 346–360)
    ParticleUniformGpu ubo{};
    std::memcpy(ubo.viewProj, scene.viewProjection.m, sizeof(ubo.viewProj));
    ubo.cameraPos[0] = scene.cameraPositionWorld.x;
    ubo.cameraPos[1] = scene.cameraPositionWorld.y;
    ubo.cameraPos[2] = scene.cameraPositionWorld.z;
    ubo.cameraPos[3] = 0.0F;
    ubo.cameraRight[0] = scene.particleCameraRight.x;
    ubo.cameraRight[1] = scene.particleCameraRight.y;
    ubo.cameraRight[2] = scene.particleCameraRight.z;
    ubo.cameraRight[3] = 0.0F;
    ubo.cameraUp[0] = scene.particleCameraUp.x;
    ubo.cameraUp[1] = scene.particleCameraUp.y;
    ubo.cameraUp[2] = scene.particleCameraUp.z;
    ubo.cameraUp[3] = 0.0F;
    std::memcpy(uniformBuffersMapped[frameIndex], &ubo, sizeof(ParticleUniformGpu));
```

### Step 4 — Match vertex attributes to GLSL locations

*VkVertexInputAttributeDescription — pos, color, size, corner.*

```cpp
// src/spark/render/sprites2d/VulkanParticlePass.cpp (lines 214–236)
    constexpr std::uint32_t kStride = sizeof(float) * kParticleFloatsPerVertex;
    VkVertexInputBindingDescription bind{};
    bind.binding = 0;
    bind.stride = kStride;
    bind.inputRate = VK_VERTEX_INPUT_RATE_VERTEX;

    VkVertexInputAttributeDescription attrs[4]{};
    attrs[0].binding = 0;
    attrs[0].location = 0;
    attrs[0].format = VK_FORMAT_R32G32B32_SFLOAT;
    attrs[0].offset = 0;
    attrs[1].binding = 0;
    attrs[1].location = 1;
    attrs[1].format = VK_FORMAT_R32G32B32A32_SFLOAT;
    attrs[1].offset = sizeof(float) * 3;
    attrs[2].binding = 0;
    attrs[2].location = 2;
    attrs[2].format = VK_FORMAT_R32_SFLOAT;
    attrs[2].offset = sizeof(float) * 7;
    attrs[3].binding = 0;
    attrs[3].location = 3;
    attrs[3].format = VK_FORMAT_R32G32_SFLOAT;
    attrs[3].offset = sizeof(float) * 8;
```

### Step 5 — Experiment — hard edge

Temporarily replace smoothstep with step in particle.frag — observe blocky quads, then revert.

### Step 6 — Experiment — oversized billboards

Double inSize in VS output debug — confirm all particles scale uniformly from center.

## 5. How This Connects to the Rest of Your Engine

**Vulkan Particle Pass** loads SPIR-V modules and sets vertex attributes. **Blend and Depth State** combines
fragment alpha with additive blend. Future **GPU Particles Roadmap** may move expansion to SSBO + procedural VS.

## 6. Pitfalls and Debugging

- **Unnormalized cameraRight/Up in UBO** — billboards skew; submit normalizes basis vectors.
- **std140 padding mismatch** — mat4 + vec4×3 needs careful alignment; compare C++ struct size to GLSL.
- **Discarding too aggressively** — edge threshold 0.004 removes faint halos; tune smoothstep range for art.
- **Corner range** — corners are ±1; size halving in VS must match artist size expectations.

## 7. Exercises

1. Tint particles blue in frag only — confirms color interpolation works.
2. Widen smoothstep band — screenshot softer vs harder edges.
3. Print `sizeof(ParticleUniformGpu)` and verify UBO allocation size.
4. Zero `cameraRight` in UBO — describe billboard collapse.
5. Change corner range from ±1 to ±0.5 — effect on disc size.

## Summary

particle.vert billboards from center + corner offset; particle.frag applies radial alpha fade; UBO supplies viewProj and camera basis each frame.

---

Next: [Blend and Depth State](06-blend-depth.md).
