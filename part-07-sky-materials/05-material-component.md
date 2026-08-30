# MaterialComponent

> **Build-Along Milestone:** Cube renders with base, normal, and ORM maps; opacity routes to transparent pass.

**You will learn:**

- Attach MaterialComponent with PBR textures and scalars on a mesh entity.
- Select LitPbr vs ToonCel and configure toon rim/band fields.
- Explain MeshDirty signaling when material parameters change.

## 1. The Problem We Are Solving

Hard-coded albedo per demo does not scale. **MaterialComponent** centralizes every parameter the scene shader
expects — textures stay on CPU until submit registers layers into `SceneRenderParams`.

Build along: add MaterialComponent beside MeshComponent on a cube, bind brick + normal + ORM, and verify submit
copies fields into SceneDrawItem.

### What is a material? (beginner)

A **mesh** answers: *what shape is it?* (triangles, vertices)  
A **material** answers: *what is it made of?* (wood, metal, glass, emissive screen)

In a game engine, a material is a **bundle of parameters** the shader reads when coloring pixels:

| Parameter | Example | Visual effect |
|-----------|---------|---------------|
| Base color / albedo | Brick texture + white tint | Surface color pattern |
| Metallic | 0.0 (wood) or 1.0 (steel) | Metals tint specular highlights |
| Roughness | 0.1 (mirror) or 0.9 (chalk) | Sharp vs blurry reflections |
| Normal map | Bumpy surface texture | Fake detail without extra geometry |
| Emissive | Glowing panel texture | Bright areas for HDR / lights |
| Opacity | 1.0 opaque, 0.5 glass | Transparent pass routing |

Spark stores these on the **CPU** in `MaterialComponent`. The GPU never sees `MaterialComponent` directly — submit flattens it into `SceneDrawItem` + texture array layers + push constants.

### Mesh + Material: two components, one entity

```text
GameObject "BrickCube"
  ├── TransformComponent     (where in the world)
  ├── MeshComponent          (which geometry — SharedPtr<Mesh>)
  └── MaterialComponent      (how it looks — textures + PBR scalars)
```

At minimum you need **Transform + Mesh** to draw something. Without `MaterialComponent`, Spark uses a flat **albedo color** from `MeshComponent` and default metallic/roughness — fine for debug gray cubes, not for production assets.

### PBR in plain language

**PBR** (Physically Based Rendering) means the shader approximates how light behaves on real surfaces:

- **Diffuse** — light scattered evenly (matte paint).
- **Specular** — shiny highlights (plastic, polished wood).
- **Metallic workflow** — metals have no diffuse; color comes from tinted reflections.

Spark's `scene.frag` implements **GGX** specular + Lambert diffuse with metallic energy conservation. You do not write that math in gameplay code — you set `metallic`, `roughness`, and textures on `MaterialComponent`; submit copies them to the GPU.

### What happens when you call SetBaseColorTexture

```text
1. You: mat->SetBaseColorTexture(brickTex);
2. MaterialComponent stores SharedPtr<Texture2D> on CPU
3. NotifyMaterialChanged() → MeshDirty signal (editor may refresh)
4. Next frame: SceneSubmit calls FindOrAddSceneTexture(brickTex)
5. Returns layer index 3 (example) — stored in SceneDrawItem.textureLayer
6. VulkanSceneTextureUploader copies RGBA pixels to array layer 3
7. Draw: push.textureLayer = 3
8. Fragment shader: texture(sceneTextures, vec3(uv, 3.0))
```

The texture is **not** uploaded when you call the setter. Upload happens during command buffer record, batched with other textures that frame. This deferred pattern avoids stalling the GPU on every inspector tweak.

## 1b. Concept Deep Dive — MaterialComponent Field Reference

### Complete private field layout

```cpp
// include/spark/ecs/components/rendering/MaterialComponent.hpp (private section)
    SharedPtr<Texture2D> baseColor, normalMap, metallicRoughness, emissiveMap, iridescenceThicknessMap;
    Vector3 tint{Vector3::One};
    float metallic = 0.0F, roughness = 0.45F;
    float metallicFactor = 1.0F, roughnessFactor = 1.0F, occlusionStrength = 1.0F;
    Vector3 emissiveColor{};
    float emissiveIntensity = 0.0F;
    Vector3 emissiveFactor{Vector3::One};
    SceneShadingModel shadingModel = SceneShadingModel::LitPbr;
    std::int32_t toonDiffuseBands = 3;
    float toonRimIntensity = 0.35F, toonRimPower = 4.0F;
    bool doubleSided = false;
    float opacity = 1.0F;
    float alphaCutoff = 0.5F;
    bool alphaBlend = false;
    MaterialUvMap baseColorUv, normalUv, metallicRoughnessUv, emissiveUv, iridescenceThicknessUv;
    float normalScale = 1.0F;
    GltfMaterialExtensions gltfExtensions;
    MaterialLibraryBinding libraryBinding;
```

Every setter calls `NotifyMaterialChanged()` → `EmitSignal(SignalId::MeshDirty)` for editor refresh.

### glTF import → MaterialComponent

```cpp
// src/spark/scene/material/GltfMaterial.cpp — ApplyTo (concept)
void GltfMaterial::ApplyTo(MaterialComponent& material) const {
    material.SetBaseColorTexture(baseColor);
    material.SetTint(baseColorFactor);
    material.SetNormalTexture(normalMap);
    material.SetNormalScale(normalScale);
    if (metallicRoughness) {
        material.SetMetallicRoughnessTexture(metallicRoughness);
        material.SetMetallic(1.0F);   // sentinel: use map channels
        material.SetRoughness(1.0F);
        material.SetMetallicFactor(metallicFactor);
        material.SetRoughnessFactor(roughnessFactor);
    }
    material.SetEmissive(emissiveColor, emissiveIntensity);
    material.SetOpacity(alphaMode == Mask ? 1.0f : baseColorFactor.w);
    material.SetGltfExtensions(extensions);
}
```

When ORM map is bound, `metallic=1.0` and `roughness=1.0` are **sentinels** telling `scene.frag` to read G/B channels from the texture instead of scalar push values.

### Submit flattening — ApplyMaterialComponentToSceneDrawItemImpl

```cpp
// src/spark/scene/submit/SceneSubmitMaterial.cpp (lines 18–45)
void ApplyMaterialComponentToSceneDrawItemImpl(SceneDrawItem& item, const MaterialComponent* mat) {
    item.metallic = mat->GetMetallic();
    item.roughness = mat->GetRoughness();
    item.shadingModel = mat->GetShadingModel();
    item.toonDiffuseBands = mat->GetToonDiffuseBands();
    item.opacity = mat->GetOpacity();
    item.baseColorUv = mat->GetBaseColorUvMap();
    item.normalUv = mat->GetNormalUvMap();
    item.metallicRoughnessUv = mat->GetMetallicRoughnessUvMap();
    item.gltfExtensions = mat->GetGltfExtensions();
    // texture layers assigned separately via FindOrAddSceneTexture
}
```

## 2. Core Concepts

### PBR parameter bundle

`MaterialComponent` stores glTF-aligned fields: base color texture + tint, normal, metallic-roughness, emissive maps, scalar metallic/roughness/factors, occlusion strength, opacity, and shading model.


### Shading model selection

`SceneShadingModel::LitPbr` vs `ToonCel` per entity — same pipeline, fragment branch. Toon fields (`toonDiffuseBands`, rim intensity/power) live on the same component.


### MeshDirty notification

Every setter calls `NotifyMaterialChanged` → `EmitSignal(MeshDirty)` so future caches/editor refresh when textures or scalars change at runtime.


### Pair with MeshComponent at submit

Submit reads optional `MaterialComponent` when gathering mesh drawables — missing material → mesh albedo only with default metallic/roughness.

### MultiMaterialComponent for glTF multi-primitive meshes

When a `Mesh` has multiple `MeshSubmesh` entries (Part VI), a single `MaterialComponent` is insufficient — each submesh may reference a different glTF material index. `MultiMaterialComponent` stores one **slot per material index**:

```cpp
// include/spark/ecs/components/rendering/MultiMaterialComponent.hpp (concept)
// Indexed by glTF material index (MeshSubmesh::materialIndex).
// Each slot: baseColor, normal, ORM, emissive textures + PBR scalars.
```

`BindFromGltfAsset` sizes slots to the imported material table and registers library keys:

```text
slot[i].materialAssetKey = "models/chair.glb#material/i"
```

At submit, `PushRigidMeshDraws` emits **one draw item per submesh**, looking up `multiMat->GetSlot(materialIndex)` for that primitive's PBR parameters. Variant materials (`KHR_materials_variants`) resolve through `MeshSubmesh::ResolveMaterialIndex(activeVariant)` before slot lookup.

**When to use which component:**

| Mesh submesh count | Component | Draw items per entity |
|--------------------|-----------|----------------------|
| 0 or 1 (full mesh) | `MaterialComponent` | 1 |
| 2+ | `MultiMaterialComponent` | 1 per submesh |

Both components store **CPU** `SharedPtr<Texture2D>` handles. GPU layer indices are assigned during submit, not at import time.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Material is its own ComponentKind

**Concept:** Separate from MeshComponent

**Decision:** Material is its own ComponentKind

**Why:** Reuse one material definition across meshes later (Part XIII serialization); swap materials without replacing geometry.

**In Spark:** `include/spark/ecs/components/rendering/MaterialComponent.hpp`

### CPU-side texture refs on component

**Concept:** SharedPtr<Texture2D> handles

**Decision:** CPU-side texture refs on component

**Why:** GPU layer indices resolved later at submit via FindOrAddSceneTexture — no VkImage in ECS.

**In Spark:** MaterialComponent texture getters

### opacity < 1 → transparentDraws partition

**Concept:** Opacity routes transparent pass

**Decision:** opacity < 1 → transparentDraws partition

**Why:** Single component drives both cutout and fade without a separate flag.

**In Spark:** `SceneSubmitDrawPartition` IsTransparentSceneDraw

### MaterialComponent(tex, tint) convenience

**Concept:** Constructor with base texture

**Decision:** MaterialComponent(tex, tint) convenience

**Why:** Quick demo spawn for textured props without chaining five setters.

**In Spark:** `MaterialComponent.cpp` constructor

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/ecs/components/rendering/MaterialComponent.hpp`
- `src/spark/ecs/components/rendering/MaterialComponent.cpp`
- `include/spark/render/scene/SceneShadingModel.hpp`

### Step 1 — Read MaterialComponent API

*PBR fields, toon params, opacity, texture accessors.*

```cpp
// include/spark/ecs/components/rendering/MaterialComponent.hpp (lines 14–97)
/**
 * Physically inspired material: optional base-color texture, tint, metallic/roughness, optional emissive texture,
 * and HDR emissive (color × intensity × emissive map RGB when a map is set).
 * Values map to the scene shader (GGX specular + Lambert diffuse with metallic energy split).
 */
class MaterialComponent final : public GameComponent {
public:
    static constexpr ComponentKind TypeKind = ComponentKind::Material;

    [[nodiscard]] ComponentKind Kind() const noexcept override { return TypeKind; }

    MaterialComponent() = default;
    MaterialComponent(SharedPtr<Texture2D> baseColor, Vector3 inTint = Vector3::One);

    void OnSignal(GameObject& owner, SignalId id, const SignalPayload& payload) override;

    [[nodiscard]] const SharedPtr<Texture2D>& GetBaseColorTexture() const noexcept { return baseColor; }
    [[nodiscard]] const SharedPtr<Texture2D>& GetNormalTexture() const noexcept { return normalMap; }
    /** glTF-style packed occlusion (R), roughness (G), metallic (B). */
    [[nodiscard]] const SharedPtr<Texture2D>& GetMetallicRoughnessTexture() const noexcept {
        return metallicRoughness;
    }
    [[nodiscard]] const SharedPtr<Texture2D>& GetEmissiveTexture() const noexcept { return emissiveMap; }
    [[nodiscard]] const Vector3& GetTint() const noexcept { return tint; }
    [[nodiscard]] float GetMetallic() const noexcept { return metallic; }
    [[nodiscard]] float GetRoughness() const noexcept { return roughness; }
    [[nodiscard]] float GetMetallicFactor() const noexcept { return metallicFactor; }
    [[nodiscard]] float GetRoughnessFactor() const noexcept { return roughnessFactor; }
    [[nodiscard]] float GetOcclusionStrength() const noexcept { return occlusionStrength; }
    [[nodiscard]] const Vector3& GetEmissiveColor() const noexcept { return emissiveColor; }
    [[nodiscard]] float GetEmissiveIntensity() const noexcept { return emissiveIntensity; }
    [[nodiscard]] const Vector3& GetEmissiveFactor() const noexcept { return emissiveFactor; }

    void SetBaseColorTexture(SharedPtr<Texture2D> tex);
    void SetNormalTexture(SharedPtr<Texture2D> tex);
    void SetMetallicRoughnessTexture(SharedPtr<Texture2D> tex);
    void SetEmissiveTexture(SharedPtr<Texture2D> tex);
    void SetTint(const Vector3& t);
    void SetMetallic(float m);
    void SetRoughness(float r);
    void SetMetallicFactor(float f);
    void SetRoughnessFactor(float f);
    void SetOcclusionStrength(float s);
    void SetEmissive(const Vector3& rgb, float intensity);
    void SetEmissiveFactor(const Vector3& f);

    [[nodiscard]] SceneShadingModel GetShadingModel() const noexcept { return shadingModel; }
    void SetShadingModel(SceneShadingModel s);
    [[nodiscard]] std::int32_t GetToonDiffuseBands() const noexcept { return toonDiffuseBands; }
    void SetToonDiffuseBands(std::int32_t bands);
    [[nodiscard]] float GetToonRimIntensity() const noexcept { return toonRimIntensity; }
    void SetToonRimIntensity(float v);
    [[nodiscard]] float GetToonRimPower() const noexcept { return toonRimPower; }
    void SetToonRimPower(float v);
    [[nodiscard]] bool IsDoubleSided() const noexcept { return doubleSided; }
    void SetDoubleSided(bool v);

    /** 1 = opaque; values below 1 route the draw to <c>SceneRenderParams::transparentDraws</c>. */
    [[nodiscard]] float GetOpacity() const noexcept { return opacity; }
    void SetOpacity(float a);

private:
    void NotifyMaterialChanged();

    SharedPtr<Texture2D> baseColor;
    SharedPtr<Texture2D> normalMap;
    SharedPtr<Texture2D> metallicRoughness;
    SharedPtr<Texture2D> emissiveMap;
    Vector3 tint{Vector3::One};
    float metallic = 0.0F;
    float roughness = 0.45F;
    float metallicFactor = 1.0F;
    float roughnessFactor = 1.0F;
    float occlusionStrength = 1.0F;
    Vector3 emissiveColor{};
    float emissiveIntensity = 0.0F;
    Vector3 emissiveFactor{Vector3::One};
    SceneShadingModel shadingModel = SceneShadingModel::LitPbr;
    std::int32_t toonDiffuseBands = 3;
    float toonRimIntensity = 0.35F;
    float toonRimPower = 4.0F;
    bool doubleSided = false;
    float opacity = 1.0F;
};
```

### Step 2 — Author material on a prop

```cpp
auto* mat = go->AddComponent<MaterialComponent>();
mat->SetBaseColorTexture(brickTex);
mat->SetTint({0.9f, 0.9f, 0.9f});
mat->SetMetallic(0.0f);
mat->SetRoughness(0.55f);
mat->SetNormalTexture(normalTex);
mat->SetMetallicRoughnessTexture(ormTex);
```

### Step 3 — Trace MeshDirty on edit

*NotifyMaterialChanged after texture/scalar setters.*

```cpp
// src/spark/ecs/components/rendering/MaterialComponent.cpp (lines 15–55)
void MaterialComponent::NotifyMaterialChanged() {
    GameObject* o = GetOwner();
    if (o != nullptr) {
        o->EmitSignal(SignalId::MeshDirty, SignalPayload{}, this);
    }
}

void MaterialComponent::SetBaseColorTexture(SharedPtr<Texture2D> tex) {
    baseColor = MoveTemp(tex);
    NotifyMaterialChanged();
}

void MaterialComponent::SetNormalTexture(SharedPtr<Texture2D> tex) {
    normalMap = MoveTemp(tex);
    NotifyMaterialChanged();
}

void MaterialComponent::SetMetallicRoughnessTexture(SharedPtr<Texture2D> tex) {
    metallicRoughness = MoveTemp(tex);
    NotifyMaterialChanged();
}

void MaterialComponent::SetEmissiveTexture(SharedPtr<Texture2D> tex) {
    emissiveMap = MoveTemp(tex);
    NotifyMaterialChanged();
}

void MaterialComponent::SetTint(const Vector3& t) {
    tint = t;
    NotifyMaterialChanged();
}

void MaterialComponent::SetMetallic(float m) {
    metallic = m;
    NotifyMaterialChanged();
}

void MaterialComponent::SetRoughness(float r) {
    roughness = r;
    NotifyMaterialChanged();
}
```

### Step 4 — Inspect SceneShadingModel enum

*LitPbr = 0, ToonCel = 1.*

```cpp
// include/spark/render/scene/SceneShadingModel.hpp (lines 7–14)
/**
 * Per-draw lit mesh shading path (see <c>shaders/scene.frag</c> + <c>VulkanRenderer::ModelPushConstants</c>).
 * <c>LitPbr</c> is Cook–Torrance; <c>ToonCel</c> uses banded diffuse, stepped specular, and rim from material fields.
 */
enum class SceneShadingModel : std::uint8_t {
    LitPbr = 0,
    ToonCel = 1,
};
```

### Step 5 — Switch to toon at runtime

```cpp
mat->SetShadingModel(SceneShadingModel::ToonCel);
mat->SetToonDiffuseBands(4);
mat->SetToonRimIntensity(0.45f);
```

Same entity, same mesh — only shading changes next submit.

### Step 6 — Milestone — textured cube with maps

Unit cube with base + normal + ORM textures registered; toggling `SetOpacity(0.5f)` moves draw to transparent partition.

## 5. How This Connects to the Rest of Your Engine

**Shading Models** consumes `shadingModel` push field. **Normal Mapping** and **ORM Textures** read layers this
component supplies. **Material Submit** calls ApplyMaterialComponentToSceneDrawItem during ECS gather.

## 6. Pitfalls and Debugging

- **Material without MeshComponent** — submit ignores orphan materials; pair on same GameObject.
- **Null texture SharedPtr** — layer stays −1; shader uses scalars only — valid, not an error.
- **Metallic=1 with dark albedo** — metals tint specular via baseColor; use light tints for gold/chrome.
- **Forgetting opacity partition** — semi-transparent draws need back-to-front sort in transparent pass.

## 7. Exercises

1. List all texture slots on `MaterialComponent` and matching `SceneDrawItem` layer fields.
2. Toggle `doubleSided` on a plane — confirm cull override in `VulkanSceneApplyRasterState`.
3. Set emissive intensity > 1 — observe HDR headroom before tonemap (Part X).
4. Call `SetOpacity(0.4f)` — trace draw into `transparentDraws` partition.
5. Swap `LitPbr` ↔ `ToonCel` on same entity without changing mesh.

## Summary

MaterialComponent owns PBR/toon parameters and texture refs; submit flattens them into SceneDrawItem layers and push constants.

---

Next: [Shading Models](06-shading-models.md).
