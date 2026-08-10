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

## 2. Core Concepts

### PBR parameter bundle

`MaterialComponent` stores glTF-aligned fields: base color texture + tint, normal, metallic-roughness, emissive maps, scalar metallic/roughness/factors, occlusion strength, opacity, and shading model.


### Shading model selection

`SceneShadingModel::LitPbr` vs `ToonCel` per entity — same pipeline, fragment branch. Toon fields (`toonDiffuseBands`, rim intensity/power) live on the same component.


### MeshDirty notification

Every setter calls `NotifyMaterialChanged` → `EmitSignal(MeshDirty)` so future caches/editor refresh when textures or scalars change at runtime.


### Pair with MeshComponent at submit

Submit reads optional `MaterialComponent` when gathering mesh drawables — missing material → mesh albedo only with default metallic/roughness.

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
