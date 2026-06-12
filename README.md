# MobileTerrainTools
### A procedural terrain editor plugin for Unreal Engine 5
*by Kims Ferdy — 2026 — v1.3*

---

## Overview

MobileTerrainTools is an editor plugin that lets you build, sculpt, paint, and ship terrain directly inside UE5 — with a full non-destructive workflow from sculpt to mobile-ready static mesh. No Landscape component, no external DCC tools required.

Designed specifically for mobile targets (Android OpenGL ES / Vulkan), but works for any platform.

---

## Current Features

### Terrain Creation — *Basic Tab*

- [x] Generate flat terrain with configurable grid (SectionsX, SectionsY, SectionSize)
- [x] Apply settings to an existing selected terrain actor
- [x] **Chunked terrain** — create a multi-chunk terrain grid (ChunksX × ChunksY) in one click
- [x] Heightmap import — load a PNG/RAW file and apply it to selected chunk
- [x] **Tiled heightmap** — apply one image across all chunks proportionally
- [x] **Heightmap edge sync** — applying a heightmap automatically syncs and smooths borders with all neighbors

---

### Sculpt Mode — *Basic Tab*

- [x] Single-chunk sculpt mode
- [x] **Global sculpt mode** — sculpts across all chunks simultaneously, brush spans chunk boundaries
- [x] Seamless chunk edge stitching — neighbor vertices stay welded during sculpting
- [x] **Cross-chunk normal stitching** — `StitchAllBorderNormals()` fires at every stroke end and grid build; border vertices use a central-difference gradient that spans both chunks, eliminating normal seams
- [x] **Edge stitching guarantee** — border heights averaged and written back to both chunks
- [x] **Brush edge falloff** — border vertices receive half brush influence
- [x] **5 sculpt tools:** Sculpt, Smooth, Flatten, Noise, Erosion brush
- [x] Circle and Square brush shapes
- [x] Brush size, strength, and falloff controls
- [x] Visual brush decal preview
- [x] Escape key exits sculpt mode cleanly

---

### Erosion — *Basic Tab*

- [x] **Real-time brush erosion** — Thermal and Hydraulic modes
- [x] **Overkill erosion** — full-terrain particle-based hydraulic simulation, background thread, non-blocking
- [x] **Multi-chunk overkill** — stitches all chunks, erodes across boundaries, splits back out

---

### Bake & Build Pipeline — *Terrain Data Tab*

- [x] Bake selected / all chunks to UStaticMesh assets
- [x] Restore selected / all chunks back to PMC
- [x] Rebake selected / all with latest sculpt data
- [x] Auto-bake after sculpt exit toggle
- [x] Live bake status indicator
- [x] **Nanite force-disabled on bake**
- [x] **Prepare for Build** — weightmap MID preserved so paint survives the SM swap
- [x] **Reverse Prepare** — actor UObject names correctly restored via `Rename()` so asset paths resolve; weightmap textures and MID fully restored
- [x] Correct runtime behavior — PIE and packaged builds use baked SM, falls back to PMC if unbaked

---

### LOD System — *Terrain Data Tab*

- [x] 5-level LOD setup applied automatically on bake
- [x] LOD presets: 2x Halving, 3x Aggressive, Custom
- [x] Per-LOD triangle percentage and screen size controls
- [x] LOD Estimator table

---

### Spline Tools — *Paint + Splines Tab*

- [x] Non-destructive — base HeightData is never modified
- [x] **5 spline types:** Road, Ramp, River, Wall, Lake
- [x] Configurable width, depth, falloff per spline
- [x] Per-point weights
- [x] Visual width guide lines in viewport
- [x] **Auto-reapply on move**
- [x] Multi-chunk support
- [x] SplineID-based storage

---

### Texture Layer Painting — *Paint + Splines Tab*

- [x] **Per-chunk weightmap texture assets** — `_WM0`, `_WM1`, ... saved under `/Textures/`
- [x] **Persistent** — real content-browser assets, survive editor restarts
- [x] **Per-chunk Material Instance Dynamic** — fully isolated parameter sets per chunk
- [x] **Real-time GPU painting** — `RHICommandList.LockTexture2D/UnlockTexture2D` streams pixels into the live GPU resource each tick; zero game thread stall
- [x] **Stroke-end flush** — Source written, `UpdateResource` called, texture package saved to disk, MID parameters pushed — all once per stroke on mouse release
- [x] **Paint persists automatically** — texture `.uasset` saved at stroke end, no manual Ctrl+S needed
- [x] **Exclusive normalized painting** — painting a layer proportionally reduces other unlocked layers; total weight sums to 1 at every vertex
- [x] Up to 4 layers per texture (RGBA); auto-expands every 4 layers
- [x] **Layer definition asset** (`UMTTLayerDefinition`) — layer names, preview colors, lock/visibility, alpha, resolution
- [x] Layer list UI — lock, visibility, color swatch, alpha slider, remove
- [x] Erase mode
- [x] Paint mode toggle — ESC exits cleanly
- [x] **Auto-fill panel on terrain selection**
- [x] **Baked SM preserves paint** — `SwapToSM` passes the MID to the static mesh component
- [x] Material parameter contract: `WeightMap0`, `WeightMap1`, ... — plugin writes them automatically
- [x] `UMTTMaterialHelper::ValidateMaterial()` — Blueprint-callable parameter checker
- [x] `UMTTMaterialHelper::DescribeLayerMapping()` — layer → channel mapping display

---

### Foliage — *Foliage Tab*

- [x] Single `AMTT_FoliageActor` per level
- [x] 8 painting tools: Select, All, Paint, Single, Fill, Erase, Remove, Move
- [x] Surface-normal-aware painting
- [x] Per-type density, scale, placement, instance, and physics settings
- [x] Overlap / min-distance check prevents stacking

---

### Data Architecture

- [x] **Organized asset folder structure:**
  ```
  Content/
  └── LevelName/
      └── Terrain/
          └── MTerrainChunk_0/
              ├── Data/      MTerrainChunk_0_Data.uasset
              ├── Meshes/    MTerrainChunk_0_SM.uasset
              └── Textures/  MTerrainChunk_0_WM0.uasset
  ```
- [x] `GetChunkAssetRoot()` — single path source of truth
- [x] `UTerrainChunkData` — HeightData, VertexColors, SplineData, WeightMapData, LOD settings, bake ref, texture refs
- [x] `UMTerrainGrid` — chunk positions and grid coordinates
- [x] `WeightMapTextures` saved (non-Transient) — references survive serialization
- [x] PIE-safe path resolution — strips `UEDPIE_0_` prefix
- [x] Cross-platform input — no Windows-only headers

---

## Bug Fixes (v1.3)

- [x] **Reverse Prepare path mismatch** (`TerrainActor_16` etc.) — `SetActorLabel` sets display name only; `GetName()` still returned the engine-assigned internal name. Fixed by calling `Rename(*ChunkLabel, nullptr, REN_DontCreateRedirectors | REN_NonTransactional)` so asset paths resolve correctly.
- [x] **Baked SM losing paint** — `SwapToSM` was passing `TerrainMaterial` (raw base) to `StaticMeshComponent`. Now passes the MID from `TerrainMeshData->GetMaterial(0)`.
- [x] **Paint not persisting across restarts** — `FlushWeightMapToGPU` now calls `UPackage::SavePackage` on each dirty texture package at stroke end.
- [x] **No real-time visual during strokes** — BulkData-only path was invisible to GPU. Replaced with `ENQUEUE_RENDER_COMMAND` + `RHICommandList.LockTexture2D/UnlockTexture2D`.
- [x] **White artifacts / wrong channel mapping** — `TSF_BGRA8` byte order mismatch. Fixed with `BgraOffset[4] = {2,1,0,3}` mapping layer 0→R, 1→G, 2→B, 3→A.
- [x] **`Source.Init` assert** (`ImageCore.cpp:1362`) — `NewObject<UTexture2D>` pre-initializes Source in UE 5.3. All three `Source.Init` call sites now guarded with `!Source.IsValid()`.
- [x] **`TSF_RGBA8` deprecated warning** — reverted to `TSF_BGRA8`, fixed channel mapping in pixel loop.
- [x] **`LoadObject` warning spam** — replaced with `FPackageName::DoesPackageExist` check; missing files handled silently.

## Bug Fixes (v1.2)

- [x] **WeightMap0 showing default texture** — `CreateMeshSection_LinearColor` reset material slot 0; MID now snapshotted before and restored after.
- [x] **Paint not marking level dirty** — `PaintWeightMap` now calls `ChunkData->Modify()` and `MarkPackageDirty()`.
- [x] **`NumTextures = 2066366080`** — `std::to_string` in `GetTextureCount()` header removed.
- [x] **Optimizer register recycle on `Def`** — `InitWeightMapTextures` reloads via `LoadSynchronous()` after soft ptr assignment.
- [x] **Prepare for Build dropping paint** — captures MID before `Destroy()`.
- [x] **Reverse Prepare ChunkData not found** — original label stored as `MTerrain_Label_X` tag.
- [x] **Reverse Prepare not restoring paint** — now calls `InitWeightMapTextures(LayerDef)`.
- [x] **All chunks sharing WeightMap0** — `EnsureMaterialInstance` checks `GetOuter() == this`.

---

## Quick Start

1. Enable: *Edit → Plugins → MobileTerrainTools*
2. Open panel: *Window → Mobile Terrain Tools*
3. **Basic tab** → set grid → **Create New Terrain**
4. **Edit Mode** → sculpt → **Escape**
5. **Paint + Splines tab** → assign Layer Asset + material with `WeightMap0` param → **Paint Mode**
6. **Terrain Data tab** → **Bake ALL to SM**
7. **Prepare for Build** before packaging

---

## Material Setup

Expose `Texture2D` parameters named `WeightMap0`, `WeightMap1`, etc. Sampler type: **Linear Color** (not sRGB).

| Parameter | R | G | B | A |
|-----------|---|---|---|---|
| WeightMap0 | Layer 0 | Layer 1 | Layer 2 | Layer 3 |
| WeightMap1 | Layer 4 | Layer 5 | Layer 6 | Layer 7 |

---

## Known Limitations (v1.3)

- [ ] **LOD cracks at chunk borders** — holes and gaps at low-poly LODs; critical fix planned for v1.4
- [ ] **Foliage tool instability** — deferred to v0.2; avoid for production use
- [ ] Corner vertices where 3+ chunks meet are not blended
- [ ] Spline point weights UI doesn't refresh when points are added/removed in viewport
- [ ] Overkill erosion is not interruptible once started
- [ ] `MaxDrawDistance` doesn't persist across bake/restore cycles

---

## Roadmap

### 🔴 Critical — v1.4
- [ ] **LOD crack/hole fix** — audit `BuildQuadLODMesh`, fix seam vertices at chunk borders, guarantee no visible gaps at any LOD level
- [ ] **Foliage tool stability audit** — document broken paths, prevent regression

### 🟡 Architecture
- [ ] **Component-based refactor** — single manager actor with terrain chunk components; matches UE landscape architecture, cleaner outliner
- [ ] **`TerrainNeighborUtils` extraction** — formalize neighbor system for sculpt seams, normal seams, world normal continuity

### 🟢 Features — v0.2
- [ ] **Brush mask system** — multiple falloff shapes + custom grayscale texture input
- [ ] **Brush surface-following** — circle follows terrain surface, reflects actual world-unit radius
- [ ] **Runtime toggle API** — strip editor tooling at runtime; runtime deformation as paid tier
- [ ] **Material layer blend modes** — per-layer opacity and blend mode; vertex color fallback
- [ ] **Runtime deformation hooks** — meteor craters, vehicle tracks
- [ ] **Layer blend transition sharpness** — per-layer control
- [ ] **Stamp tool** — place a height stamp from a texture
- [ ] **Mirror sculpt mode** — X / Y / XY symmetry

### 📋 Polish / QoL
- [ ] Spline tool: dropdown instead of hardcoded buttons
- [ ] Save All Terrain Data button
- [ ] Persist brush settings across editor restarts (`GConfig`)
- [ ] Chunk grid visualizer overlay
- [ ] Plugin settings panel (default size, material, layer resolution)
- [ ] Actual triangle counts in LOD estimator
- [ ] Runtime LOD distance tuning from panel

### 🌐 Platform Verification
- [ ] UE 5.1 / 5.3 / 5.6 / 5.7 verified builds
- [ ] Mac/Linux compilation verified

---

*MobileTerrainTools v1.3 — UE 5.5 — © Kims Ferdy 2026*
