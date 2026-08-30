# Clustered Forward

> **Build-Along Milestone:** CPU builds a 16×16×16 cluster grid and uploads light SSBOs; `scene.frag` evaluates all punctual lights when `ubo.clusterDepth.w` is enabled.

**You will learn:**

- Explain clustered forward shading vs brute-force and vs deferred.
- Build Spark's 3D screen-depth cluster grid on the CPU.
- Upload `ClusterLightsGpu` and `ClusterGridGpu` to SSBO bindings 4 and 5.
- Trace the punctual light loops in `accumulatePunctualPbr`.

## 1. The Problem We Are Solving

Forward rendering with **N lights × M pixels** naively costs O(lights × fragments). At 256 point lights and a million fragments, that hurts. **Clustered forward** (also called forward+) assigns lights to **3D clusters** of screen space (XY tiles × depth slices) so each fragment only considers lights overlapping its cluster.

Spark implements the **full CPU cluster builder** and SSBO infrastructure — grid metadata in UBO, lights in SSBO binding 4, indices in binding 5. The current `scene.frag` path **iterates all punctual lights** per fragment (acceptable at 256+128 cap) with a comment that cluster lookup is reserved for a future tiled path. This chapter teaches the architecture Spark has and the shader loop it runs today.

### Why not loop all 256 lights per pixel? (beginner)

Naive forward shading:

```glsl
for (int i = 0; i < pointLightCount; ++i)
  color += shadePoint(params.pointLights[i]);
```

With 200 lights and 2M pixels, that's 400M light evaluations per frame — too slow on mobile GPUs.

**Clustered lighting** divides view space into a grid (e.g. 16×16×16 cells). Each cell stores a **small list** of light indices that overlap that volume. Each pixel only loops lights in its cell:

```text
View frustum divided into clusters (like 3D buckets)
  Cluster (5,3,12) → lights [2, 7, 41]
  Pixel in that cluster → loop 3 lights, not 256
```

**Spark today:** CPU builds the cluster grid and uploads SSBOs; `scene.frag` may still loop all punctual lights until tiled lookup is enabled — the data structures are ready.

**Beginner mental model:**

| Approach | Cost |
|----------|------|
| All lights per pixel | O(pixels × all lights) |
| Clustered | O(pixels × lights in cell) |

Cells with no lights → cheap. Dense city night scene → more lights per cell but still bounded.

## 1b. Concept Deep Dive — Cluster Grid and SSBO Layout

### 16×16×16 grid on CPU

`VulkanClusteredForwardLights.cpp` assigns each point/spot light to screen-depth clusters:

```cpp
// Cluster index: ix + iy * kClusterGridX + iz * (kClusterGridX * kClusterGridY)
std::uint32_t ClusterIndex(const int ix, const int iy, const int iz);

int FindDepthSlice(const float dist, const float nearPlane, const float farPlane, const int gridZ);
// logarithmic depth slicing between near and far clip planes
```

Lights project to NDC, expand by screen-radius, and mark every overlapping tile — a conservative halo prevents boundary pixels from missing influencers.

### UBO metadata + SSBO payload

```cpp
// SceneUniformGpu cluster fields (written by VulkanSceneUniformWriter)
float clusterGrid[4]{};   // xyz = grid dims; w = point light count
float clusterDepth[4]{};  // z = spot count; w = clustered enabled flag
```

SSBO binding 4 holds packed `ClusterLightsGpu` records; binding 5 holds per-cluster index lists (`ClusterGridGpu`). Shaders decode light type from index ranges using counts in the UBO.

### Current shader path vs infrastructure

Spark **builds and uploads** the full cluster grid every frame. `scene.frag` may still iterate all punctual lights in `accumulatePunctualPbr` until tiled lookup is enabled — the SSBO data is production-ready; the inner loop is the incremental optimization. Profiling at 256+128 lights is when cluster lookup pays off.

### Perspective Y flip alignment

`NdcToScreenTile` flips clip Y to match `gl_FragCoord` and `sparkClusterIndex` in GLSL — mismatched Y convention assigns lights to wrong horizontal tiles (symptom: lights appear only on half the screen).

## 2. Core Concepts

### Cluster grid dimensions

```cpp
constexpr uint32_t kClusterGridX = 16;
constexpr uint32_t kClusterGridY = 16;
constexpr uint32_t kClusterGridZ = 16;
constexpr uint32_t kClusterCount = 4096;  // 16³
```

Each cluster is a frustum slice:

- **X/Y** — screen divided into 16×16 tiles (NDC → screen → tile index)
- **Z** — logarithmic depth slices between near and far planes

```text
Screen space (16×16 tiles) × depth (16 slices) = 4096 clusters
```

### Logarithmic depth slices

```cpp
int FindDepthSlice(float dist, float nearPlane, float farPlane, int gridZ) {
    float t = log(dist / nearPlane) / log(farPlane / nearPlane);
    return floor(t * gridZ);
}
```

More slices near camera, fewer far away — matches perspective depth distribution.

### Per-light cluster assignment (CPU)

For each point/spot light, `VulkanClusteredForwardLights`:

1. Project light position to NDC
2. Estimate screen radius from range
3. Compute tile AABB (`NdcToScreenTile`)
4. Compute depth slice range from light position ± range
5. Insert light index into every overlapping cluster's list

Conservative halo expands tile bounds by one pixel — reduces boundary misses.

### SSBO layouts

**Binding 4 — lights list:**

```glsl
layout(std430, set = 0, binding = 4) readonly buffer ClusterLightsSSBO {
    uint numPointLights;
    uint numSpotLights;
    vec4 pointPositionRange[256];
    vec4 pointColorIntensity[256];
    vec4 spotPositionRange[128];
    vec4 spotDirectionCosOuter[128];
    vec4 spotColorIntensity[128];
    vec4 spotCosInner[128];
} clusterLights;
```

**Binding 5 — cluster indices:**

```glsl
layout(std430, set = 0, binding = 5) readonly buffer ClusterGridSSBO {
    uint clusterOffsets[4096];
    uint clusterCounts[4096];
    uint clusterIndices[16384];
} clusterGrid;
```

C++ mirrors: `ClusterLightsGpu`, `ClusterGridGpu` with `static_assert` sizes.

### UBO cluster metadata

```glsl
vec4 clusterGrid;  // xyz = grid dims, w = num point lights (decode aid)
vec4 clusterDepth; // xy = near/far, z = num spots, w = enabled flag
```

CPU write in `VulkanSceneUniformWriter`:

```cpp
u.clusterGrid[0..2] = 16, 16, 16;
u.clusterGrid[3] = static_cast<float>(numPointLights);
u.clusterDepth[0] = clusterNear;
u.clusterDepth[1] = clusterFar;
u.clusterDepth[2] = static_cast<float>(numSpotLights);
u.clusterDepth[3] = (npl > 0 || nsl > 0) ? 1.0f : 0.0f;
```

Fragment early-out when `ubo.clusterDepth.w < 0.5` — no punctual work.

### Current shader evaluation path

```glsl
void accumulatePunctualPbr(...) {
    if (ubo.clusterDepth.w < 0.5) return;
    for (uint i = 0u; i < numPt; ++i) {
        // point: atten + evalBRDF + optional shadow
    }
    for (uint si = 0u; si < numSp; ++si) {
        // spot: evalSpotBrdfFromData + shadow
    }
}
```

**Does not yet index `clusterGrid` by fragment cluster ID** — comment in `clustered_lights.glsl`: *"Cluster grid kept for future tiled path."* CPU still builds grid so tooling and future optimization are ready.

### Why build grid anyway?

- Validates light assignment logic before shader complexity
- Enables future `clusterIndex = f(gl_FragCoord, depth)` lookup
- Debugging: visualize cluster counts per tile
- Sprite pass may share infrastructure

### Performance characteristics today

| Lights | Cost per fragment |
|--------|-------------------|
| 0 | Early-out, cheap |
| 32 | 32 × evalBRDF |
| 256 point + 128 spot | 384 BRDF max |

Fine for desktop teaching demo; cluster lookup needed for hundreds+ lights at 4K.

### BuildAndUpload frame hook

```cpp
VulkanClusteredForwardLights::BuildAndUpload(
    scene, resolvedLighting, viewProj, cameraPos, frameIndex);
```

- Packs light arrays from `SceneRenderParams`
- Fills cluster grid lists
- Memcpy to mapped SSBOs for frame slot

Descriptor binding 4–5 updated or persistent mapped buffers per frame-in-flight.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### 16³ grid resolution

**Concept:** Cluster dimensions

**Decision:** 4096 clusters, max 128 lights per cluster

**Why:** Balance CPU build cost vs granularity; matches common forward+ literature.

**In Spark:** `VulkanClusteredLightGpu.hpp`.

### CPU cluster build

**Concept:** Per-light assignment

**Decision:** Build on CPU each frame, not compute shader

**Why:** Simpler bootstrap; light count fits CPU loop; no async compute sync.

**In Spark:** `VulkanClusteredForwardLights.cpp`.

### Direct SSBO scan in fragment shader

**Concept:** Current evaluation path

**Decision:** Loop all lights until cluster lookup lands

**Why:** Correct lighting at current caps; ship book Part IV without tiled shader.

**In Spark:** `accumulatePunctualPbr` in `scene.frag`.

### clusterDepth.w enable gate

**Concept:** UBO metadata

**Decision:** Single float toggles punctual path

**Why:** Sun-only scenes skip 384-iteration loops entirely.

**In Spark:** `VulkanSceneUniformWriter` + shader guard.

### Shared attenuation helpers

**Concept:** SSBO layouts

**Decision:** `sparkPointAttenuation`, `sparkSpotAngularAttenuation` in include

**Why:** Sprite and scene passes share math.

**In Spark:** `clustered_lights.glsl`.

## 4. Implementation Walkthrough

### Spark Files We Touch This Chapter

- `include/spark/render/scene/VulkanClusteredForwardLights.hpp`
- `src/spark/render/scene/VulkanClusteredForwardLights.cpp`
- `include/spark/render/scene/VulkanClusteredLightGpu.hpp`
- `shaders/clustered_lights.glsl`
- `shaders/scene.frag` (`accumulatePunctualPbr`)

### Step 1 — Sun only

Zero point/spot lights — `clusterDepth.w = 0`, punctual loops skipped. Profiler shows no SSBO iteration cost.

### Step 2 — One point light

Place near center. Verify SSBO `numPointLights = 1`, mesh brightens, falloff with distance.

### Step 3 — Many lights stress

64 point lights — still correct if slow; confirms O(lights) shader path.

### Step 4 — Inspect cluster grid CPU

Log max `clusterCounts[i]` — no cluster should exceed 128 (`kMaxLightsPerCluster`).

### Step 5 — Future cluster lookup sketch

Pseudocode for tiled path:

```glsl
uint clusterIdx = sparkClusterIndex(gl_FragCoord, gl_FragCoord.z, ubo);
uint offset = clusterGrid.clusterOffsets[clusterIdx];
uint count = clusterGrid.clusterCounts[clusterIdx];
for (uint j = 0; j < count; ++j) {
    uint lightId = clusterGrid.clusterIndices[offset + j];
    // evaluate lightId
}
```

Compare to current full scan.

### Step 6 — Toon punctual

`accumulatePunctualToon` mirrors PBR loop with stepped diffuse — same SSBO data.

## 5. How This Connects to the Rest of Your Engine

Chapter **7–8** fill SSBO source data. Chapter **6** `evalBRDF` evaluates each hit. Part **V** adds punctual shadow sampling inside loops. Part **IX** may enable true cluster lookup or move build to compute.

Clustered forward is Spark's **scalability story** — infrastructure now, optimization later.

## 6. Pitfalls and Debugging

- **No punctual despite lights** — `clusterDepth.w == 0` or SSBO not bound.
- **SSBO layout mismatch** — validation or garbage positions.
- **Lights wrong after resize** — cluster build uses stale viewProj.
- **Spot in wrong cluster** — cone radius estimate too small; halo fixes.
- **Assuming cluster culling works** — full scan still runs; profile accordingly.

## 7. Exercises

1. Compute worst-case BRDF calls per pixel with max lights.
2. Visualize `clusterCounts` as heatmap (CPU readback debug).
3. Implement fragment `clusterIndex` from `gl_FragCoord` and log it for center pixel.
4. When does CPU cluster build exceed 1 ms with 200 lights? (measure)
5. Compare clustered forward to deferred in one paragraph — tradeoffs.

## Summary

Spark uses a 16×16×16 cluster grid built on the CPU each frame, uploading light data to SSBO binding 4 and cluster indices to binding 5. UBO fields `clusterGrid` and `clusterDepth` describe the grid and enable punctual shading. Today's `scene.frag` iterates all point and spot lights when enabled — correct at Spark's caps, with cluster lookup ready for future optimization.

---

Next: [Lighting Profiles](10-time-of-day.md).
