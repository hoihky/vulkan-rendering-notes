# Skeleton and Bones

> **Build-Along Milestone:** Bind-pose skin palette evaluated for imported character with ≤64 joints.

**You will learn:**

- Represent joint hierarchy, inverse bind matrices, and animation clips on CPU.
- Evaluate world joint matrices and build skin palette entries.
- Relate Skeleton::MaxJoints to glTF import limits.

## 1. The Problem We Are Solving

Skinned meshes deform with a **bone hierarchy** and bind-pose inverses. Without a clear CPU skeleton type,
animation clips cannot target joints and the GPU cannot blend vertices.

Build along: import a skinned glTF (next chapter), inspect joint count and clip names, evaluate bind-pose palette.

## 2. Core Concepts

### Joint hierarchy

A **skeleton** is a tree of joints: parent indices, rest local transforms, and **inverse bind matrices** from glTF skin. Animation clips store sparse TRS key channels targeting joint indices.


### Skin matrix definition

GPU palette entry: `skinMatrix[j] = jointGlobalPrefix[j] * worldJoint[j] * inverseBind[j]`. Vertices stay in bind pose on CPU; shader multiplies by the blended skin matrix.


### Topological evaluation

`ComputeJointWorldMatrices` resolves parent chains — parents must be ready before children. Spark uses iterative `SolveWorldMatrix` with a done[] guard for out-of-order joint lists.


### MaxJoints = 64

`Skeleton::MaxJoints` caps palette size for SSBO packing and push constant jointCount. glTF skins with >64 joints fail import early.


### Clip sampling

`SampleClipPose` lerps translation/scale, slerps rotation per channel at clamped time. Missing channels fall back to `restLocal` bind pose.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Skeleton class with jointParents + inverseBind

**Concept:** Joint hierarchy

**Decision:** Skeleton class with jointParents + inverseBind

**Why:** Single CPU type holds hierarchy, clips, and palette builders — Animator reads SharedPtr<Skeleton>.

**In Spark:** `include/spark/animation/Skeleton.hpp`

### BuildPaletteFromPose

**Concept:** Skin matrix definition

**Decision:** BuildPaletteFromPose

**Why:** Centralizes bind-space correction via jointGlobalPrefix from glTF scene graph.

**In Spark:** `Skeleton::BuildPaletteFromPose` — Skeleton.cpp

### ComputeJointWorldMatrices

**Concept:** Topological evaluation

**Decision:** ComputeJointWorldMatrices

**Why:** Handles unsorted glTF joint arrays without requiring strict topological import order.

**In Spark:** `Skeleton.cpp` — SolveWorldMatrix helper

### Static cap at import and GPU

**Concept:** MaxJoints = 64

**Decision:** Static cap at import and GPU

**Why:** Matches skin SSBO sizing and scene.vert jointCount guard.

**In Spark:** `Skeleton::MaxJoints` + skinned_mesh_gltf jointCount check

## 4. Implementation Walkthrough

Follow these steps in order. You are extending the same mini-engine throughout the book;
Spark is the reference implementation you can build, run, and diff against.

### Spark Files We Touch This Chapter

- `include/spark/animation/Skeleton.hpp`
- `src/spark/animation/Skeleton.cpp`
- `src/spark/scene/skinned_mesh_gltf.cpp`

### Step 1 — Read Skeleton public API

*MaxJoints, clip queries, SampleClipPose, BuildPaletteFromPose.*

```cpp
// include/spark/animation/Skeleton.hpp (lines 18–50)
/**
 * Joint hierarchy + inverse bind matrices + sampled glTF animations.
 * Palettes are computed in bind/character space for the vertex shader (world = model * skinnedPos).
 */
class Skeleton {
public:
    static constexpr std::uint32_t MaxJoints = 64;

    Skeleton() = default;

    [[nodiscard]] std::uint32_t GetJointCount() const noexcept { return jointCount; }
    [[nodiscard]] std::uint32_t GetClipCount() const noexcept { return clipNames.GetSize(); }
    [[nodiscard]] float GetClipDuration(std::uint32_t clipIndex) const;
    [[nodiscard]] const Utf8String& GetClipName(std::uint32_t clipIndex) const;
    /** Returns -1 when no clip matches (case-sensitive UTF-8 name). */
    [[nodiscard]] std::int32_t FindClipIndexByName(const char* name) const;
    /** Case-insensitive exact name match; returns -1 when no clip matches. */
    [[nodiscard]] std::int32_t FindClipIndexByNameCaseInsensitive(const char* name) const;
    /** First clip whose name contains <c>substring</c> (case-insensitive); returns -1 when none match. */
    [[nodiscard]] std::int32_t FindClipIndexIfNameContains(const char* substring) const;

    /**
     * Fills the first jointCount entries: skinMatrix[j] = worldJoint[j] * inverseBind[j].
     * clipIndex must be < GetClipCount(); timeSec wraps with fmod (legacy sampling).
     */
    void ComputePalette(std::uint32_t clipIndex, float timeSec, Matrix4* outPalette, std::uint32_t paletteMax)
            const;

    /** Samples clip pose at timeSec (clamped to [0, duration], no wrap). */
    void SampleClipPose(std::uint32_t clipIndex, float timeSec, Array<Transform>& outPose) const;

    /** Builds skin palette from a local joint pose (size must be jointCount). */
    void BuildPaletteFromPose(const Array<Transform>& pose, Matrix4* outPalette, std::uint32_t paletteMax) const;
```

### Step 2 — Trace SampleClipPose

*Rest pose + translation/scale/rotation channel sampling.*

```cpp
// src/spark/animation/Skeleton.cpp (lines 208–244)
void Skeleton::SampleClipPose(std::uint32_t clipIndex, float timeSec, Array<Transform>& outPose) const {
    if (jointCount == 0 || clipIndex >= clips.GetSize()) {
        outPose.Clear();
        return;
    }
    const AnimationClip& clip = clips[clipIndex];
    const float dur = clip.duration > 0.0F ? clip.duration : 1.0e-4F;
    float t = timeSec;
    if (t < 0.0F) {
        t = 0.0F;
    }
    if (t > dur) {
        t = dur;
    }

    outPose.Resize(jointCount);
    for (std::uint32_t j = 0; j < jointCount; ++j) {
        outPose[j] = restLocal[j];
    }

    for (std::size_t ci = 0; ci < clip.translations.GetSize(); ++ci) {
        const Vec3Channel& ch = clip.translations[ci];
        if (ch.jointIndex < jointCount) {
            outPose[ch.jointIndex].translation = SampleVec3Linear(ch.times, ch.values, t);
        }
    }
    for (std::size_t ci = 0; ci < clip.scales.GetSize(); ++ci) {
        const Vec3Channel& ch = clip.scales[ci];
        if (ch.jointIndex < jointCount) {
            outPose[ch.jointIndex].scale = SampleVec3Linear(ch.times, ch.values, t);
        }
    }
    for (std::size_t ci = 0; ci < clip.rotations.GetSize(); ++ci) {
        const QuatChannel& ch = clip.rotations[ci];
        if (ch.jointIndex < jointCount) {
            outPose[ch.jointIndex].rotation = SampleQuatSlerp(ch.times, ch.values, t);
        }
```

### Step 3 — Follow world matrix evaluation

*ComputeJointWorldMatrices — parent chain resolution.*

```cpp
// src/spark/animation/Skeleton.cpp (lines 189–206)
void Skeleton::ComputeJointWorldMatrices(
        std::uint32_t jc,
        const Array<std::int32_t>& parents,
        const Array<Matrix4>& locals,
        Array<Matrix4>& outWorld) {
    if (jc == 0) {
        return;
    }
    outWorld.Resize(jc);
    Array<bool> done;
    done.Resize(jc);
    for (std::uint32_t j = 0; j < jc; ++j) {
        done[j] = false;
    }
    for (std::uint32_t j = 0; j < jc; ++j) {
        SolveWorldMatrix(j, parents, locals, outWorld, done);
    }
}
```

### Step 4 — Build skin palette

*locals → world → prefix * world * inverseBind.*

```cpp
// src/spark/animation/Skeleton.cpp (lines 248–269)
void Skeleton::BuildPaletteFromPose(
        const Array<Transform>& pose,
        Matrix4* outPalette,
        std::uint32_t paletteMax) const {
    if (outPalette == nullptr || jointCount == 0 || paletteMax < jointCount || pose.GetSize() < jointCount) {
        return;
    }

    Array<Matrix4> locals;
    locals.Resize(jointCount);
    for (std::uint32_t j = 0; j < jointCount; ++j) {
        locals[j] = pose[j].ToMatrix4();
    }

    Array<Matrix4> world;
    ComputeJointWorldMatrices(jointCount, jointParents, locals, world);

    const bool usePrefix = jointGlobalPrefix.GetSize() == jointCount;
    for (std::uint32_t j = 0; j < jointCount; ++j) {
        const Matrix4& pre = usePrefix ? jointGlobalPrefix[j] : Matrix4::Identity;
        outPalette[j] = pre * world[j] * inverseBind[j];
    }
```

### Step 5 — Inspect glTF skin import cap

*Reject skins with zero or > MaxJoints joints.*

```cpp
// src/spark/scene/skinned_mesh_gltf.cpp (lines 541–546)
    cgltf_skin* skin = skinNode->skin;
    const std::uint32_t jointCount = static_cast<std::uint32_t>(skin->joints_count);
    if (jointCount == 0 || jointCount > Skeleton::MaxJoints) {
        cgltf_free(data);
        return false;
    }
```

### Step 6 — Debug single joint rotation

After import, call `SampleClipPose` at t=0 and t=0.5; log one limb joint translation delta to verify channels target expected indices.

### Step 7 — Milestone check — bind pose palette

At clip time 0, skin matrices should deform minimally (rest shape). Log joint 0 matrix vs identity for a T-pose character.

## 5. How This Connects to the Rest of Your Engine

**Skinned glTF** fills Skeleton during `TryLoadSkinnedCharacterFromGltf`. **Animator** samples clips into
local pose then calls BuildPaletteFromPose. **GPU Skinning** consumes the palette in scene.vert.

## 6. Pitfalls and Debugging

- **Wrong joint order** — parent indices must match glTF skin.joints array order.
- **Missing inverse bind** — mesh collapses or explodes; verify inverseBindMatrices accessor loaded.
- **>64 joints** — import fails; simplify skeleton or raise cap coherently in C++ and GLSL.
- **Euler rotation keys** — Spark slerps quaternions; ensure glTF rotations are quaternion channels.

## 7. Exercises

1. Print joint count and first three clip names after skinned import.
2. Sample clip at t=0 and t=1; compare root joint translation.
3. Manually set one local rotation in pose; verify one limb moves in palette debug draw.

## Summary

Skeleton stores joint tree, inverse binds, and clips; BuildPaletteFromPose outputs GPU skin matrices.

---

Next: [Skinned glTF](07-skinned-gltf.md).
