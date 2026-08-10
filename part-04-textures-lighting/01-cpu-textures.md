# Loading CPU Textures

> **Build-Along Milestone:** A `Texture2D` holds decoded RGBA8 pixels on the CPU; `SceneRenderParams::sceneTextures` collects shared layers ready for GPU upload.

**You will learn:**

- Store texture pixels in a renderer-agnostic CPU type.
- Load PNG and JPEG into RGBA8 with stb_image.
- Deduplicate textures into a scene texture array with stable layer indices.
- Choose vertical flip conventions for file vs embedded image data.

## 1. The Problem We Are Solving

Gameplay, editors, and asset loaders should not call `vkCreateImage`. Art arrives as files on disk, procedural generators, or embedded bytes inside glTF — all **CPU-side** until the render frame commits a snapshot.

Spark inserts `Texture2D` between assets and Vulkan:

```text
Disk / glTF bytes  →  Texture2D (RGBA8)  →  SceneRenderParams::sceneTextures  →  GPU uploader
```

If this boundary is weak — mixed formats, no deduplication, flip chaos — every downstream chapter (upload, sampling, lighting) becomes unreliable. This chapter makes the CPU contract explicit: **always RGBA8**, named layers, fingerprinted for change detection, max 16 entries per frame.

## 2. Core Concepts

### Texture2D: the CPU pixel container

`Texture2D` lives in `spark/scene/` and includes **no Vulkan headers**:

```cpp
class Texture2D {
    std::uint32_t width, height;
    Array<std::uint8_t> rgba;           // w * h * 4 bytes
    std::uint64_t contentFingerprint; // FNV-1a hash
};
```

| Field | Purpose |
|-------|---------|
| `width`, `height` | Pixel dimensions (any positive size) |
| `rgba` | Interleaved RGBA8, row-major, top row first after decode |
| `contentFingerprint` | Invalidates GPU upload when pixels change |
| `name` | Debug label (`Utf8String`) |

GPU upload may **resample** to 512×512 per layer — CPU dimensions need not match atlas tile size. This decouples art resolution from GPU memory layout.

### RGBA8 as the universal internal format

Spark normalizes all decode paths to **four 8-bit channels**:

| Source | Decoder | Output |
|--------|---------|--------|
| PNG/JPEG on disk | `stbi_load(..., 4)` | RGBA8 |
| glTF embedded buffer | `stbi_load_from_memory(..., 4)` | RGBA8 |
| Procedural (`CreateCheckerboard`) | CPU fill | RGBA8 |

No BC7, no HDR `.exr`, no single-channel R8 in the scene path — simplicity over format breadth for the teaching book. Part XVI discusses production format expansion.

### stb_image integration

Implementation lives in one translation unit to avoid duplicate symbols:

```cpp
// src/spark/scene/stb_image_impl.cpp
#define STB_IMAGE_IMPLEMENTATION
#include "stb_image.h"
```

`TryLoadFromFile` decodes and transfers ownership to `Texture2D`:

```cpp
bool Texture2D::TryLoadFromFile(const char* path, Texture2D& out, bool flipVerticalOnLoad) {
    stbi_set_flip_vertically_on_load(flipVerticalOnLoad ? 1 : 0);
    unsigned char* data = stbi_load(path, &w, &h, nullptr, 4);
    if (!data) return false;
    // copy into Array<uint8_t>, stbi_image_free(data)
    out.SetPixels(w, h, MoveTemp(bytes));
    return true;
}
```

**Default `flipVerticalOnLoad = true`** for legacy file loads. **Memory/glTF path uses `flip = 0`** because row 0 must be the texture top for Vulkan upload without an extra flip in shader UVs.

### Content fingerprinting

`SetPixels` recomputes FNV-1a over width, height, and raw bytes:

```cpp
void Texture2D::RefreshContentFingerprint() noexcept {
    std::uint64_t h = Fnv64Begin();
    Fnv64Mix(h, width);
    Fnv64Mix(h, height);
    h = Fnv64HashBytes(h, rgba.GetData(), rgba.GetSize());
    contentFingerprint = h;
}
```

`VulkanSceneTextureUploader::NeedsUpload` compares fingerprints per layer — editing pixels in place without `SetPixels` would **not** trigger re-upload. Always go through `SetPixels` or reload.

### Scene texture array in SceneRenderParams

```cpp
// SceneRenderParams.hpp
static constexpr std::size_t MaxSceneTextures = 16;
Array<SharedPtr<Texture2D>> sceneTextures;
```

Each `SceneDrawItem` stores `textureLayer` — an index into this array (`-1` = no texture sample, tint only). The same array holds albedo, normal map, metallic-roughness, emissive, and sky equirect layers — different draws index different slots.

Registration during scene submit:

```cpp
// SceneSubmitMaterial.cpp — FindOrAddSceneTexture
for (std::size_t i = 0; i < params.sceneTextures.GetSize(); ++i) {
    if (params.sceneTextures[i].Get() == tex.Get()) return static_cast<std::int32_t>(i);
}
params.sceneTextures.PushBack(tex);
return static_cast<std::int32_t>(params.sceneTextures.GetSize() - 1);
```

Pointer equality deduplicates — two draws sharing the same `SharedPtr<Texture2D>` get the same layer index.

### Procedural fallbacks

When assets are missing, demos stay runnable:

| Helper | Use |
|--------|-----|
| `CreateCheckerboard` | Debug UV visualization |
| `CreateSolid` | Flat color materials |
| `CreateBrickPattern` | Wall fallback |
| `CreateSoilPattern` / `CreateVariedSoilGroundPattern` | Terrain albedo |

Procedural textures are first-class `Texture2D` objects — same upload path as file loads.

### CPU resampling before GPU

`ResampleNearest` and `ResampleBilinear` resize on CPU. The uploader calls **nearest** to 512² — fast, acceptable for distant props; bilinear available when quality matters on CPU before upload.

## 3. Design Decisions

Each decision below references a concept from Section 2 and shows where Spark applies it.

### Texture2D in scene/, not render/

**Concept:** Separation from Vulkan

**Decision:** Zero `vulkan.h` in texture type

**Why:** Gameplay and ECS can load art without linking GPU code; snapshot stays platform-agnostic.

**In Spark:** `include/spark/scene/Texture2D.hpp`.

### Always RGBA8 internal

**Concept:** RGBA8 universal format

**Decision:** Force 4 channels in every `stbi_load` call

**Why:** One upload format (`R8G8B8A8_UNORM`), one shader sample path, predictable memory size.

**In Spark:** `Texture2D.cpp` all decode paths.

### Fingerprint-driven GPU invalidation

**Concept:** Content fingerprinting

**Decision:** Hash pixels, not file path mtime

**Why:** Procedural edits, runtime painting, and glTF memory loads have no file timestamp.

**In Spark:** `VulkanSceneTextureUploader::NeedsUpload`.

### Pointer-deduped sceneTextures array

**Concept:** Scene texture array

**Decision:** `FindOrAddSceneTexture` by `SharedPtr` address

**Why:** Same image used on 50 draws → one GPU layer, one index.

**In Spark:** `SceneSubmitMaterial.cpp`.

### Different flip defaults per load path

**Concept:** stb_image vertical flip

**Decision:** File load flip-on; memory load flip-off

**Why:** glTF and Vulkan agree row 0 = top; legacy file art may expect stb default. Document per path, do not guess.

**In Spark:** `TryLoadFromFile` vs `TryLoadFromMemory` comments in `Texture2D.cpp`.

## 4. Implementation Walkthrough

Follow these steps in order. You are building the asset side of the texture pipeline; GPU upload is Chapter 2.

### Spark Files We Touch This Chapter

- `include/spark/scene/Texture2D.hpp`
- `src/spark/scene/Texture2D.cpp`
- `src/spark/scene/stb_image_impl.cpp`
- `include/spark/engine/SceneRenderParams.hpp`
- `src/spark/scene/SceneSubmitMaterial.cpp`

### Step 1 — Load a PNG

```cpp
Texture2D albedo;
if (!Texture2D::TryLoadFromFile("assets/brick.png", albedo)) {
    albedo = Texture2D::CreateBrickPattern(256, 256);
}
```

Log `GetWidth()`, `GetHeight()`, `GetContentFingerprint()`. Reload same file — fingerprint should match if pixels unchanged.

### Step 2 — Build sceneTextures for one frame

```cpp
SceneRenderParams params;
params.sceneTextures.Clear();
SharedPtr<Texture2D> shared = MakeShared<Texture2D>(MoveTemp(albedo));
params.sceneTextures.PushBack(shared);
// drawItem.textureLayer = 0;
```

### Step 3 — Test deduplication

Push the same `SharedPtr` twice through `FindOrAddSceneTexture` — array size stays 1, index 0 returned both times.

### Step 4 — Flip experiment

Load the same PNG with `flipVerticalOnLoad true` vs `false`. Preview in a CPU image viewer or after GPU upload (Chapter 2) — one orientation will match your mesh UVs.

### Step 5 — Procedural fallback path

Delete the PNG path temporarily. Confirm `CreateBrickPattern` fills `sceneTextures[0]` and demo still runs.

### Step 6 — Fingerprint mutation

Call `SetPixels` with modified bytes. Assert `GetContentFingerprint()` changes — uploader will detect on next frame (Chapter 2).

## 5. How This Connects to the Rest of Your Engine

Chapter **2** uploads `sceneTextures` to a `VkImage` 2D array. Chapter **3** binds that image to descriptor binding 1. Chapter **4** maps mesh UVs to texel addresses. Chapter **5** decodes sRGB when sampling albedo.

The CPU layer is the **source of truth** for pixel content until the GPU copy completes.

## 6. Pitfalls and Debugging

- **Upside-down texture after upload** — wrong `flipVerticalOnLoad` for your UV convention; fix at load, not in shader when possible.
- **Stale GPU after pixel edit** — mutated `rgba` without `SetPixels` → fingerprint unchanged.
- **Layer index -1 but shader samples anyway** — guard `vTextureLayer >= 0` in fragment shader.
- **More than 16 textures** — `MaxSceneTextures` exceeded; merge atlases or evict layers.
- **stb duplicate symbol link error** — `STB_IMAGE_IMPLEMENTATION` in exactly one `.cpp`.

## 7. Exercises

1. Load a JPEG and PNG of the same image — compare fingerprints after decode (should match if pixels match).
2. Implement `CreateGradientTexture(width, height, topColor, bottomColor)`.
3. Count unique `sceneTextures` entries in a demo scene with 20 draws — how many layers after dedup?
4. Log fingerprint before/after `ResampleNearest` to 512² — does resample belong in fingerprint? (Spark hashes pre-resample CPU source.)
5. Write a table: which Spark demos call `sceneTextures.Clear()` each frame and why.

## Summary

`Texture2D` stores RGBA8 pixels on the CPU with FNV-1a fingerprints for change detection. stb_image loads files and memory buffers; flip convention differs per path. `SceneRenderParams::sceneTextures` holds up to 16 deduplicated layers referenced by `textureLayer` on each draw. Keep textures free of Vulkan types so the render snapshot stays a clean handoff to the GPU uploader.

---

Next: [GPU Texture Upload](02-gpu-upload.md).
