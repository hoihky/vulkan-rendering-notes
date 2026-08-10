# Normal Mapping

> **Build-Along Milestone:** Normal map toggled live; specular detail follows ripples without mesh subdivision.

**You will learn:**

- Build TBN and perturb world normal from a tangent-space normal map.
- Register `normalMapLayer` at submit from `MaterialComponent`.
- Use derivative fallback when tangents are missing or degenerate.

## 1. The Problem We Are Solving

Flat meshes need **surface detail** without extra geometry. Normal maps perturb shading normals per texel — but wrong TBN, sRGB decode, or green-channel convention makes lighting slide across UV seams or look mushy.

Build along: assign a normal texture on `MaterialComponent`, toggle it in `MaterialShowcase3DDemo` (**2** key), and compare specular footprint on the high-segment sphere.

## 2. Core Concepts

### Tangent-space perturbation

Normal maps store surface detail as RGB offsets in tangent space (typically Z-up in map space). The shader builds a **TBN** matrix and transforms sampled perturbation to world **N** before any lighting dot products.

```text
N_world = normalize(TBN * (sample * 2 - 1))
```

| TBN column | Source |
|------------|--------|
| T | Vertex tangent (normalized) |
| B | `cross(N, T) * tangent.w` (handedness) |
| N | Interpolated geometric normal |

### normalMapLayer index

When `push.normalMapLayer >= 0`, fragment shader samples `sceneTextures` array layer. `−1` skips map and uses interpolated vertex normal only.

### Handedness fix

Spark flips `tN.y = -tN.y` after `* 2 - 1` decode — common OpenGL vs DirectX normal map green channel convention mismatch. If lighting inverts when you import from Substance, toggle this line first.

### Derivative TBN fallback

If vertex tangents are degenerate (`dot(vTangent, vTangent) < ε`), `dFdx`/`dFdy` of world position and UV reconstruct **T** and **B** — costs fragment bandwidth but saves broken glTF imports missing tangents.

### MikkTSpace and glTF

glTF assets baked with MikkTSpace produce tangents that match standard normal maps. Spark's glTF loader (Part VI) preserves tangent vec4 on vertices; procedural meshes from `CreateSkySphere` may rely on derivative fallback.

### Data vs color texture

Normal maps are **linear data** — no `sparkSrgbToLinear` on sample. Upload as `VK_FORMAT_R8G8B8A8_UNORM` (or similar), not sRGB.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### FindOrAddSceneTexture for normal map

**Concept:** Register layer at submit

**Decision:** Dedup by `SharedPtr` identity

**Why:** Same rules as albedo — stable layer index across draws.

**In Spark:** `SceneSubmitMaterial.cpp` `ApplyMaterialComponentToSceneDrawItem`

### No sRGB decode on normal sample

**Concept:** Linear normal data

**Decision:** Raw `.xyz` from texture

**Why:** Only albedo uses `sparkSrgbToLinear`.

**In Spark:** `scene.frag` normal map block

### vTangent vec4 with w handedness

**Concept:** Vertex tangent carry

**Decision:** Location 3 in vertex layout

**Why:** Standard MikkTSpace **w** sign for bitangent cross product.

**In Spark:** `scene.vert` tangent output

### BuildTangentNormalRipples CPU texture

**Concept:** Procedural normal in showcase

**Decision:** Demo generates ripples when no asset file

**Why:** Validates mapping without external glTF.

**In Spark:** `MaterialShowcase3DDemo.cpp` `BuildTangentNormalRipples`

## 4. Implementation Walkthrough

### Spark Files We Touch This Chapter

- `shaders/scene.frag`
- `shaders/scene.vert`
- `src/spark/scene/SceneSubmitMaterial.cpp`
- `src/spark/demo/MaterialShowcase3DDemo.cpp`

### Step 1 — Read normal mapping block

*ORM fetch then TBN construction and normal perturbation.*

```glsl
// shaders/scene.frag (lines 449–479)
    vec3 nGeom = normalize(vNormal);
    vec3 N = nGeom;
    if (push.normalMapLayer >= 0) {
        int nl = clamp(push.normalMapLayer, 0, 15);
        vec3 tN = texture(sceneTextures, vec3(vTexCoord, float(nl))).xyz * 2.0 - 1.0;
        tN.y = -tN.y;
        mat3 TBN;
        if (dot(vTangent.xyz, vTangent.xyz) > 1e-8) {
            vec3 T = normalize(vTangent.xyz);
            vec3 B = normalize(cross(nGeom, T) * vTangent.w);
            TBN = mat3(T, B, nGeom);
        } else {
            vec3 dp1 = dFdx(vWorldPos);
            vec3 dp2 = dFdy(vWorldPos);
            vec2 duv1 = dFdx(vTexCoord);
            vec2 duv2 = dFdy(vTexCoord);
            float det = duv1.x * duv2.y - duv1.y * duv2.x;
            if (abs(det) > 1e-8) {
                float invDet = 1.0 / det;
                vec3 Tu = (dp1 * duv2.y - dp2 * duv1.y) * invDet;
                vec3 Tv = (dp2 * duv1.x - dp1 * duv2.x) * invDet;
                vec3 T = normalize(Tu - nGeom * dot(nGeom, Tu));
                float handedness = (det < 0.0) ? -1.0 : 1.0;
                vec3 B = normalize(cross(nGeom, T) * handedness);
                TBN = mat3(T, B, nGeom);
            } else {
                TBN = mat3(vec3(1.0, 0.0, 0.0), vec3(0.0, 1.0, 0.0), nGeom);
            }
        }
        N = normalize(TBN * tN);
    }
```

### Step 2 — Wire normal texture on material

```cpp
mat->SetNormalTexture(normalTex);
```

Submit sets `item.normalMapLayer` via `ApplyMaterialComponentToSceneDrawItem` with `&params` overload.

### Step 3 — Trace submit registration

```cpp
// src/spark/scene/SceneSubmitMaterial.cpp
    if (mat->GetNormalTexture()) {
        item.normalMapLayer = SceneSubmitDetail::FindOrAddSceneTexture(*resolveTextures, mat->GetNormalTexture());
    }
```

Base color layer is resolved separately in the drawable sink — normal does not share albedo's layer index.

### Step 4 — Toggle normal map in showcase

Run `MaterialShowcase3DDemo` — press **2** to toggle normal map. Specular highlights should shrink into ripple valleys when enabled.

### Step 5 — Debug view (temporary shader edit)

```glsl
// outColor = vec4(N * 0.5 + 0.5, 1.0); return;
```

Verify continuity across UV seams on the sphere. Seams that flash indicate bad tangents on import.

### Step 6 — Milestone — ripples visible in specular

Sphere with normal map shows highlight movement vs flat normal — rotate point light to confirm detail is in shading, not geometry.

## 5. How This Connects to the Rest of Your Engine

**ORM Textures** samples additional layers in the same fetch preamble. **Shading Models** use perturbed **N** for both PBR and toon. **Part VI** glTF import must preserve tangents for best quality. **Part IV** sun and punctual lights all dot against the same **N**.

## 6. Pitfalls and Debugging

- **sRGB on normal texture** — soft, wrong lighting; upload as linear UNORM.
- **Flipped green channel** — invert lighting; toggle `tN.y` flip to match authoring tool.
- **Non-orthogonal TBN after interpolation** — reconstruct per fragment or Gram-Schmidt.
- **normalMapLayer stale** — texture array order changes if dedup fails; use `SharedPtr` identity.
- **Flat normal map (128,128,255)** — no visible change; verify texture not empty.

## 7. Exercises

1. Disable normal map layer (`−1`) — screenshot before/after on brick sphere.
2. Flip G channel in texture — observe lighting inversion.
3. Visualize T, B, N as RGB on a cube face.
4. Zero vertex tangents on a glTF mesh — confirm derivative fallback still shows detail.
5. Count extra texture samples per fragment when normal + ORM + albedo all bound.

## Summary

Normal maps perturb **N** via TBN from `normalMapLayer`; Spark decodes linear tangent data with Y flip and derivative fallback when tangents are missing.

---

Next: [ORM Textures](08-orm-textures.md).
