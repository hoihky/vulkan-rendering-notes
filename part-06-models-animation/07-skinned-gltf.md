# Skinned glTF

> **Build-Along Milestone:** Khronos skinned character spawns with SkinnedMeshComponent and renders in bind pose.

**You will learn:**

- Import SkinnedGltfAsset with mesh, skeleton, clips, and bind alignment.
- Wire SkinnedMeshComponent and AnimatorComponent on one GameObject.
- Explain jointPalette fill during SceneSubmit skinned gather.

## 1. The Problem We Are Solving

Importing skinned glTF is not enough — ECS must connect mesh, skeleton, and animator so submit builds draw items
the renderer recognizes as **skinned** rather than rigid.

Build along: load a Khronos sample character, spawn the component trio, verify bind-pose render on Custom slot.

## 2. Core Concepts

### SkinnedMesh vertex layout

Extends rigid attributes with **four joint indices and weights**, plus optional tangent for normal maps. Weights should sum to 1; importer normalizes when glTF data drifts.


### TryLoadSkinnedCharacterFromGltf

Scans scene for first skinned node, loads JOINTS_0/WEIGHTS_0 accessors, builds Skeleton clips from glTF animations, and returns walk clip heuristic + bind alignment quaternions for gameplay facing.


### ECS component trio

`SkinnedMeshComponent` holds geometry; `AnimatorComponent` holds SharedPtr<Skeleton> and playback state; `MaterialComponent` supplies PBR textures — same as rigid props.


### Submit skinned draw items

`SkinnedSubmitSink` fills `SceneDrawItem.skinnedMesh`, resizes `jointPalette`, calls `anim->ComputeJointPalette` **during gather** so GPU sees current-frame matrices.


### Bind alignment offsets

SkinnedGltfAsset carries `bindUpAlignment` and `bindFacingYawOffset` — CharacterCameraDemo rotates visual root so varied glTF bind poses stand upright on the ground plane.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### SkinnedMesh::Vertex with joints[4] + weights[4]

**Concept:** SkinnedMesh vertex layout

**Decision:** SkinnedMesh::Vertex with joints[4] + weights[4]

**Why:** Matches scene.vert packed joint fetch; AppendRigidMeshAsSkinned upgrades rigid for testing.

**In Spark:** `include/spark/scene/SkinnedMesh.hpp`

### Separate from rigid TryLoadFromGltf

**Concept:** TryLoadSkinnedCharacterFromGltf

**Decision:** Separate from rigid TryLoadFromGltf

**Why:** Keeps skin weights out of rigid Mesh path; one function returns mesh + skeleton + clips.

**In Spark:** `src/spark/scene/skinned_mesh_gltf.cpp`

### SkinnedMeshComponent + AnimatorComponent

**Concept:** ECS component trio

**Decision:** SkinnedMeshComponent + AnimatorComponent

**Why:** Scene submit requires both mesh and animator; missing either skips skinned draw.

**In Spark:** `SkinnedSubmitSink::OnSkinnedDrawable` — SceneSubmit.cpp

### jointPalette on SceneDrawItem

**Concept:** Submit skinned draw items

**Decision:** jointPalette on SceneDrawItem

**Why:** Palette computed at submit — not stored on component — avoids stale GPU data if tick order wrong.

**In Spark:** `item.jointPalette.Resize(jc)` + ComputeJointPalette

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/scene/SkinnedMesh.hpp`
- `src/spark/scene/skinned_mesh_gltf.cpp`
- `src/spark/scene/SceneSubmit.cpp`
- `src/spark/demo/CharacterCameraDemo.cpp`

### Step 1 — Read SkinnedMesh vertex

*Position, normal, UV, tangent, four bone influences.*

```cpp
// include/spark/scene/SkinnedMesh.hpp (lines 17–31)
/**
 * Mesh with up to 4 bone influences per vertex (glTF JOINTS_0 / WEIGHTS_0).
 * Vertices are stored in bind pose; GPU applies joint palette * inverse bind.
 */
class SkinnedMesh {
public:
    struct Vertex {
        Vector3 position{Vector3::Zero};
        Vector3 normal{Vector3::UnitZ};
        Vector2 texCoord{Vector2::Zero};
        /** xyz = tangent direction, w = bitangent handedness; zero xyz = derivative TBN fallback. */
        Vector4 tangent{};
        std::uint32_t joints[4]{0, 0, 0, 0};
        float weights[4]{1.0F, 0.0F, 0.0F, 0.0F};
    };
```

### Step 2 — Open skinned import entry

*Parse glTF, find skinned node, validate joint count.*

```cpp
// src/spark/scene/skinned_mesh_gltf.cpp (lines 472–546)
bool TryLoadSkinnedCharacterFromGltf(
        const char* path,
        SkinnedMesh& outMesh,
        Skeleton& outSkeleton,
        SharedPtr<Texture2D>* outBaseColor,
        std::uint32_t* outWalkClipIndex,
        Quaternion* outBindUpAlignment,
        float* outBindFacingYawOffset) {
    if (path == nullptr || path[0] == '\0') {
        return false;
    }
    if (outWalkClipIndex != nullptr) {
        *outWalkClipIndex = 0;
    }
    if (outBindUpAlignment != nullptr) {
        *outBindUpAlignment = Quaternion::Identity;
    }
    if (outBindFacingYawOffset != nullptr) {
        *outBindFacingYawOffset = 0.0F;
    }

    cgltf_options options{};
    cgltf_data* data = nullptr;
    if (cgltf_parse_file(&options, path, &data) != cgltf_result_success || data == nullptr) {
        return false;
    }
    if (cgltf_load_buffers(&options, data, path) != cgltf_result_success) {
        cgltf_free(data);
        return false;
    }

    outMesh.Clear();
    outMesh.GetName() = Utf8String(path);
    outSkeleton.jointCount = 0;
    outSkeleton.jointParents.Clear();
    outSkeleton.inverseBind.Clear();
    outSkeleton.restLocal.Clear();
    outSkeleton.clips.Clear();
    outSkeleton.clipNames.Clear();
    outSkeleton.jointGlobalPrefix.Clear();

    cgltf_scene* primary = data->scene;
    if (primary == nullptr && data->scenes_count > 0) {
        primary = &data->scenes[0];
    }

    cgltf_node* skinNode = nullptr;
    if (primary != nullptr) {
        for (cgltf_size i = 0; i < primary->nodes_count; ++i) {
            ScanForSkinnedMeshNode(primary->nodes[i], &skinNode);
            if (skinNode != nullptr) {
                break;
            }
        }
    }
    if (skinNode == nullptr) {
        for (cgltf_size si = 0; si < data->nodes_count; ++si) {
            ScanForSkinnedMeshNode(&data->nodes[si], &skinNode);
            if (skinNode != nullptr) {
                break;
            }
        }
    }

    if (skinNode == nullptr || skinNode->mesh == nullptr || skinNode->skin == nullptr) {
        cgltf_free(data);
        return false;
    }

    cgltf_skin* skin = skinNode->skin;
    const std::uint32_t jointCount = static_cast<std::uint32_t>(skin->joints_count);
    if (jointCount == 0 || jointCount > Skeleton::MaxJoints) {
        cgltf_free(data);
        return false;
    }
```

### Step 3 — Load bundle from cache

*LoadSkinnedGltf → SharedPtr mesh + skeleton + walkClipIndex.*

```cpp
// src/spark/scene/GameWorldAssetCache.cpp (lines 97–123)
SkinnedGltfAsset GameWorldAssetCache::LoadSkinnedGltf(const char* path) {
    SkinnedGltfAsset empty{};
    if (path == nullptr || path[0] == '\0') {
        return empty;
    }
    const Utf8String key(path);
    if (const SkinnedGltfAsset* cached = skinnedGltfCache.Find(key)) {
        return *cached;
    }
    SkinnedMesh mesh;
    Skeleton skeleton;
    SharedPtr<Texture2D> tex;
    std::uint32_t walkClip = 0;
    Quaternion bindUp{};
    float facingYaw = 0.0F;
    if (!TryLoadSkinnedCharacterFromGltf(path, mesh, skeleton, &tex, &walkClip, &bindUp, &facingYaw)) {
        return empty;
    }
    SkinnedGltfAsset asset{};
    asset.mesh = SharedPtr<SkinnedMesh>(new SkinnedMesh(MoveTemp(mesh)));
    asset.skeleton = SharedPtr<Skeleton>(new Skeleton(MoveTemp(skeleton)));
    asset.baseColorTexture = tex;
    asset.walkClipIndex = walkClip;
    asset.bindUpAlignment = bindUp;
    asset.bindFacingYawOffset = facingYaw;
    skinnedGltfCache.Add(key, asset);
    return asset;
```

### Step 4 — Spawn character components

*SkinnedMeshComponent + Character3DAnimFsm + AnimatorComponent + Material.*

```cpp
// src/spark/demo/CharacterCameraDemo.cpp (lines 943–958)
    if (characterVisual == nullptr) {
        characterVisual = characterRoot->GetWorld().CreateGameObject();
        characterVisual->GetName() = Spark::Utf8String("PlayerVisual");
        characterVisual->SetParent(characterRoot);
        characterVisualTr = characterVisual->AddComponent<Spark::TransformComponent>();
        characterSkinnedMesh = characterVisual->AddComponent<Spark::SkinnedMeshComponent>(asset.mesh);
        charAnimFsm = characterVisual->AddComponent<Spark::Character3DAnimFsmComponent>();
        playerAnimator = characterVisual->AddComponent<Spark::AnimatorComponent>(
                asset.skeleton, asset.walkClipIndex, 1.0F);
        if (asset.baseColorTexture) {
            characterMaterial = characterVisual->AddComponent<Spark::MaterialComponent>(
                    asset.baseColorTexture, Spark::Vector3::One);
            characterMaterial->SetMetallic(0.0F);
            characterMaterial->SetRoughness(0.55F);
        } else {
            characterMaterial = characterVisual->AddComponent<Spark::MaterialComponent>();
```

### Step 5 — Trace submit palette fill

*SkinnedSubmitSink — skinnedMesh, jointPalette, ComputeJointPalette.*

```cpp
// src/spark/scene/SceneSubmit.cpp (lines 288–319)
        void OnSkinnedDrawable(GameObject* /*object*/,
                const SkinnedMeshComponent& smc,
                const MaterialComponent* mat,
                const AnimatorComponent* anim,
                const Matrix4& world) override {
            if (!smc.GetMesh() || anim == nullptr || !anim->GetSkeleton()) {
                return;
            }
            const std::uint32_t jc = anim->GetSkeleton()->GetJointCount();
            if (jc == 0) {
                return;
            }
            SceneDrawItem item{};
            item.model = world;
            item.mesh = SceneMeshSlot::Custom;
            item.skinnedMesh = smc.GetMesh();
            item.albedo = {0.9F, 0.88F, 0.82F};
            item.textureLayer = -1;
            item.metallic = 0.0F;
            item.roughness = 0.5F;
            if (mat != nullptr) {
                ApplyMaterialComponentToSceneDrawItem(item, mat, &params);
                if (mat->GetBaseColorTexture()) {
                    const Vector3& t = mat->GetTint();
                    item.albedo = {item.albedo.x * t.x, item.albedo.y * t.y, item.albedo.z * t.z};
                    item.textureLayer = findTex(mat->GetBaseColorTexture());
                }
            }
            item.jointPalette.Resize(jc);
            anim->ComputeJointPalette(item.jointPalette.GetData(), Skeleton::MaxJoints);
            item.shadowFlags = defaultShadowFlags;
            draws.PushBack(item);
```

### Step 6 — Apply bind alignment

*bindUpAlignment + yaw offset on character root rotation.*

```cpp
// src/spark/demo/CharacterCameraDemo.cpp (lines 866–941)
    humanModelBindFix = asset.bindUpAlignment.Normalized();
    humanModelYawOffset = isFox ? -Spark::HalfPi : asset.bindFacingYawOffset;
    rig.characterFacingYawOffset = humanModelYawOffset;
    rig.characterRootBindOrientation = humanModelBindFix;

    Spark::Vector3 alignedMin{};
    Spark::Vector3 alignedMax{};
    bool haveBounds = false;
    float minAlignY = 0.0F;
    {
        const Spark::Array<Spark::SkinnedMesh::Vertex>& sv = asset.mesh->GetVertices();
        if (!sv.IsEmpty()) {
            haveBounds = true;
            bool first = true;
            for (std::size_t vi = 0; vi < sv.GetSize(); ++vi) {
                const Spark::Vector3 p = humanModelBindFix.RotateVector(sv[vi].position);
                if (first) {
                    alignedMin = p;
                    alignedMax = p;
                    minAlignY = p.y;
                    first = false;
                } else {
                    alignedMin.x = std::min(alignedMin.x, p.x);
                    alignedMin.y = std::min(alignedMin.y, p.y);
                    alignedMin.z = std::min(alignedMin.z, p.z);
                    alignedMax.x = std::max(alignedMax.x, p.x);
                    alignedMax.y = std::max(alignedMax.y, p.y);
                    alignedMax.z = std::max(alignedMax.z, p.z);
                    minAlignY = std::min(minAlignY, p.y);
                }
            }
        }
    }

    constexpr float kTargetHeightM = 1.75F;
    float sc = 0.04F;
    if (haveBounds) {
        const float h = std::max(1.0e-4F, alignedMax.y - alignedMin.y);
        sc = kTargetHeightM / h;
    }

    Spark::Vector3 visualLocalOffset{};
    if (haveBounds) {
        characterVisualFootOffsetY = -minAlignY * sc;
        visualLocalOffset = {
                -(alignedMin.x + alignedMax.x) * 0.5F * sc,
                characterVisualFootOffsetY,
                -(alignedMin.z + alignedMax.z) * 0.5F * sc};
    } else {
        characterVisualFootOffsetY = 0.0F;
    }

    if (haveBounds) {
        const float H = (alignedMax.y - alignedMin.y) * sc;
        rig.firstPersonEyeHeight = std::clamp(H * 0.88F, 1.4F, 1.95F);
        rig.firstPersonForwardNudge = std::clamp(H * 0.08F, 0.14F, 0.42F);
        rig.thirdPersonPivotHeight = std::clamp(H * 0.88F, 1.32F, 1.9F);
        rig.thirdPersonFocusAhead = std::clamp(H * 0.05F, 0.12F, 0.38F);
        rig.thirdPersonCameraLift = std::clamp(H * 0.12F, 0.35F, 0.68F);
    } else {
        rig.firstPersonEyeHeight = 1.68F;
        rig.firstPersonForwardNudge = 0.28F;
        rig.thirdPersonPivotHeight = 1.55F;
        rig.thirdPersonFocusAhead = 0.32F;
        rig.thirdPersonCameraLift = 0.5F;
    }

    characterAvatarHudName = isFox ? Spark::Utf8String("Khronos Fox")
                                   : Spark::Utf8String("Khronos CesiumMan robot");

    characterRootTr->SetUniformScale(1.0F);
    characterRootTr->SetRotation(
            (Spark::Quaternion::FromAxisAngle(
                     Spark::Vector3::UnitY, rig.characterVisualYaw + humanModelYawOffset)
             * humanModelBindFix)
                    .Normalized());
```

### Step 7 — Configure locomotion clips from skeleton

*ConfigureLocomotionFromSkeleton after spawn.*

```cpp
// src/spark/demo/CharacterCameraDemo.cpp (lines 976–981)
    if (charAnimFsm != nullptr) {
        charAnimFsm->ClearManualClip();
        charAnimFsm->ConfigureLocomotionFromSkeleton(*asset.skeleton, asset.walkClipIndex);
        charAnimFsm->SetWalkSpeedThreshold(0.35F);
        charAnimFsm->SetRunSpeedThreshold(2.5F);
    }
```

### Step 8 — Milestone check — skinned character in bind pose

Load Khronos CesiumMan or Fox `.glb`, spawn with three components, see bind-pose mesh on ground before animation tick (clip index valid, palette identity-ish).

## 5. How This Connects to the Rest of Your Engine

**Animator** advances clips into palettes consumed here at submit. **GPU Skinning** deforms vertices in scene.vert.
**Locomotion FSM** selects which clip Animator plays on the same entity.

## 6. Pitfalls and Debugging

- **Rigid path for skinned mesh** — joints ignored; mesh stays bind pose; need skinnedMesh on draw item.
- **Animator missing at submit** — SkinnedSubmitSink returns early; no draw.
- **Palette computed after submit** — one-frame lag; compute during gather only.
- **Ignoring bind alignment** — character floats or lies on side; apply SkinnedGltfAsset orientation hints.

## 7. Exercises

1. Disable AnimatorComponent; confirm submit skips or bind pose only.
2. Log walkClipIndex and clip count from LoadSkinnedGltf.
3. Compare joint count to glTF skin.joints length in external viewer.

## Summary

Skinned glTF bundles mesh+skeleton; ECS components + submit jointPalette connect animation to Custom pool draws.

---

Next: [Animation Clips](08-animator.md).
