# 03 - UV handling, islands, baking, export, AO and de-lighting

Paths: `F/` = `Assets/_gm/Features/`, `C/` = `Assets/_gm/_Core/`. Reuse marks as in README §1.

## 1. UV handling: what exists and what does not

* **No auto-unwrap, no packing, no island repair.** Grep over the whole `Assets/` for `unwrap`, `xatlas`, `GenerateSecondaryUVSet`, `uvatlas` finds nothing. SPZ textures
  whatever UVs the imported mesh carries. If a mesh has no UVs it falls back to a single empty tile (`UDIMs_Helper`, "sometimes models have no UVs. Oct 2024 version 2.0.2").
  Overcast's "automatic UVs" feature therefore has **no SPZ counterpart**; nothing to take for unwrap.
* **Island handling** is limited to a **chunk mask**: a texture that is 1 inside any UV triangle and 0 elsewhere, rendered once per import/resolution change
  (`F/Render/Objects_Renderer_MGR.cs RenderUVChunksTex_maybe` with `F/TextureTools/UV Chunks/Color_UV_Chunks.shader`; a sibling `Fill_UV_Chunks.shader` is used by bucket fill). It
  drives dilation (`02`), brush gutter handling (`05`), and the "what is covered" test. Overcast already has a UV coverage notion; the useful point is to keep it
  **separate from colour alpha** and at a stable resolution.

## 2. UDIM support (worth checking our edge cases against)

Files: `F/TextureTools/UDIMs/UDIMs_Helper.cs`, `RenderUdims.cs`, `C/Shader_Includes/UDIM_AutoDefines.cginc`, `TextureArrays_AutoDefines.cginc`.

| Rule | Detail | Mark |
|---|---|---|
| Tile index from UV | `tile = max(0, ceil(coord) - 1)` per axis. Comment: "users had uvs as 1.0 and 1.0 and expected it to still be in first usual udim. Therefore always Ceil, then subtract 1. Don't Floor." Negative UVs clamp into tile 0. | ALGO |
| Tile id | `1001 + x + 10*y`; `UDIM_Sector(int)` inverts it. | ALGO |
| Looping UV into [0,1] | `loopUV`: for `uv<0`: `frac(min(uv+1e-6, 0))`, else `frac(max(0, uv-1e-6))`. The comment: plain `frac` glitches when UVs sit exactly on tile borders (0.0, 1.0, 3.0). So the value is pulled 1e-6 into the tile. | ALGO |
| Scanning meshes | Per mesh on a background `Task`, at most 8 meshes in flight; results unioned into a hash set, then sorted by `(y, x)` ("Important, else {1011,1003} can happen", fixed Aug/Sep 2024). | IDEA |
| Storage | **One slice of a `Texture2DArray` per UDIM** (all slices same resolution). Hard cap **24** (`MAX_NUM_UDIMS`); shaders are compiled in variants for ≤8, ≤16, ≤24 slices. | IDEA (the cap is an engine artefact) |
| Triangle → slice | Taken from the **first vertex's UV only** (`uv_to_renderTargIX(g[0].uv)`). A triangle straddling two tiles lands in the first vertex's tile. | pitfall, see `12` |
| Texture import | Single texture → tile (0,0). Several → parse a 4-digit number 1000-2000 from the filename (scanning from the end); if any fails, assign 1001, 1002, … in dictionary order (`Determine_UDIMs`, `Extract_UDIM_fromPath`). | ALGO |
| Export naming | `EncodeAndSaveTextures` appends the UDIM suffix only when there is more than one texture (or a flag). | IDEA |

`ALGO` checklist for Overcast: unit-test tile index and UV looping with values `-0.5, 0.0, 1e-7, 1.0, 1.0000001, 2.0, 3.0`; keep triangle→tile selection per triangle (centroid or majority vertex) rather than first vertex.

## 3. Mesh import and normalisation

* Importer: AssimpNet(ter) at runtime (`F/3D Models/AssimpLoader.cs`). Post-process flags: `Triangulate | GenerateSmoothNormals | CalculateTangentSpace | MakeLeftHanded | FlipWindingOrder | JoinIdenticalVertices`
  (UV flip explicitly *not* applied: "Unity UV is already bottom-left"). `uMesh.indexFormat = UInt32` so large meshes work.
* **Scale normalisation**: `Objs3D_Container.RescaleModel_fitIntoVolume` resets rotation/scale/position, takes total renderer bounds, scales uniformly so that the largest extent is **3.0 units**
  and re-centres the bounds on the origin. It stores `currModelRoot_scaleAfterImport` and warns if the original max extent was > 1001 units ("distant light in the FBX" will break auto-depth).
  Rationale in the README: "Culling planes are fixed (0.25-1000) for float precision". This is why constants like the depth bias `3e-5` work across models.
  `IDEA` for Overcast: if depth bias or tolerances are expressed relative to the model's bounding box, we never need a global scale; otherwise do the same normalisation inside the render path only (do not mutate the user's mesh).
* Mesh identity: every sub-mesh gets a `ushort unique_id` (`SD_3D_Mesh`, `Objs3D_Container.meshID_to_mesh`), used for selection, picking (ID render, `09`), and as stable keys in the project file.
* Per-mesh selection decides which meshes a projector writes to (`ProjectorCamera._myMeshes`, `EquipMaterial_on_Specific`).

## 4. Baking / compositing to the atlas

"Baking" is simply the **per-frame accumulation texture** (`_accumulation_uv_RT`, RGBA8, default 2048 per tile, Bilinear) described in `02` §2. There is no separate offline bake; the
exact same texture is what the viewport samples and what export saves.

Resolution ladder (`F/Settings/SceneResolution_MGR.cs`): 32, 48, 64, 96, 128, 192, 256, 384, 512, 768, 1024, 2048, 3072, 4096, 5120, 6144, 7168, 8192 (stepped by +/- buttons; low values exist for pixel-art looks with point filtering).
Interesting resource-management rules found there: resolution is **temporarily raised to 5k during generation** ("to avoid blurring when inpainting") and **kept at 4k when saving a project**
(the project-save path memorises and restores the previous resolution). Overcast can mimic: project at a higher internal resolution than the viewport texture while generating tiles.

Export (`F/Save Load Import Export/Save_MGR.cs`):

* `Get_ProjectionsDict(isDilate)`: if dilating, `numIters = max(w,h)/16`, rule per `02` §3, channel A, `bordersWiderBlur = true`, run synchronously. Then each array slice becomes a `Texture2D`.
* `EncodeAndSaveTextures`: PNG/JPG/TGA per UDIM, AO exported alongside as `<name>_AO`.
* `Export3D_with_textures`: saves the **original imported mesh bytes** back to a file (`ModelsHandler_3D.ExportModel` → `SaveCachedMesh_toFile`), plus textures named after it. The mesh itself is never rewritten, so UVs are untouched; an FBX helper exists
  (`ModelsHandler_SaveFBX_Helper.cs`, Autodesk FBX SDK via Unity package) for the built-in default door and conversions. `IDEA`: for a Blender bridge, "textures change, mesh bytes stay" keeps round-trips safe.
* `Save_MGR.MergeIcons`: renders the composite, hands the baked albedo textures to a caller-supplied callback (which turns them into a new flat layer) and, unless `oldIcons_survive` is set, disposes all UV-texture, UV-brush and projection generations. A "flatten layers" operation worth offering with an explicit confirmation (SPZ has no undo for it, see `04`).
* `Save_Mesh_Textures` always waits 3 frames (`WaitForRenderAll_crtn`) after forcing a re-render, to be sure all cameras and passes have completed. A numpy pipeline can be synchronous.

## 5. Ambient occlusion bake into UV (useful, simple)

Files: `F/TextureTools/AO/AmbientOcclusion_Baker.cs`, `SSAO_Bake.shader`, `AO_BlitApply.shader`.

Idea: **render the model from many random directions and accumulate screen-space AO into the UV atlas** with an exponential moving average.

```
repeat ~1024 iterations (two random view directions per iteration; optional bias towards "from below")
    render depth of the model from direction dir (random in unit sphere)          # 2D screen-space depth
    for each visible UV texel (rasterised into UV):
        ao_view = 1 - strength * mean_k smoothstep(0, fullDiff, depth - depth(sample_k)) over 16 hemisphere samples,
                  samples = radius/depth * reflect(sphere_k, random_vec(noiseTex)), flipped into the normal hemisphere
        ao      = prev * stab + ao_view * (1 - stab)           # stab default 0.97 in the shader, 0.99 on the baker component
    texel not visible this iteration: keep prev
blur inside UV chunks (3 iterations preview, 5-8 final), dilate, store as R (G = 1 marks island coverage)
```

* Parameters found: radius range 0.03-0.5 (default 0.1), full effect depth difference 0.05-0.4, strength 0-6, depth bias `3e-5`, preview texture is 1/4 resolution, minimum final resolution 2048.
* Apply (`AO_BlitApply.shader`): a three-band curve around a pivot: `dark/mid/light = smoothstep` masks; `ao' = isDark·(ao + darks − 1) + isMid·(ao + mids − 1) + isLight·(ao + lights − 1)`; then `lerp(1, ao', visibility)`; written to RGB and multiplied over the accumulated colour (the accumulation is cleared to white when an AO layer exists).
* Mark `ALGO`, effort S-M in a rasteriser we already have (we have per-view visibility and a UV raster; a CPU/OpenCV version needs a depth buffer per direction, which our headless renderer already produces for visibility). Value for Overcast: AO as a **multiplicative shading layer** can hide residual lighting inconsistencies between tiles and add depth to flat generated textures. Medium value; not on the critical path.
* Quality tip from the code: accumulate more iterations rather than a bigger kernel; blur only inside chunks (`BlurInsideUVchunks.shader` skips samples whose alpha is 0, so colour never bleeds in from outside the island; the "ray" steps are exaggerated with `lerp(1, boxHalf, farSteps)` to get a large cheap blur).

## 6. De-lighting (pattern-aware shading mask)

Files: `F/TextureTools/Delight/PatternAwareDelighter.compute`, `Delight_MGR.cs`, `ShadowR_PythonRunner.cs`.

* `PatternAwareDelighter.compute` is a **heuristic shading detector**: convert to CIE-LAB; luminance deviation from L=50 and chroma deviation are weighted
  (0.7 / 0.3); texture "patterns" (frequency bands along 4 directions over 8 px, left/right and up/down symmetry, edge counts, gradient coherence on rings, 32-px periodicity) are
  scored and **subtracted**, so repeating texture is not mistaken for lighting; output is a single-channel "shading likelihood" mask. Compensation then pulls L toward 50 while keeping 20 % of local contrast, with a +20 % chroma boost where the mask is high.
  Observed: `Delight_MGR.cs` is full of `//MODIF` debug markers and a test-only `Update`; it looks experimental. `IDEA` only; do not rely on it. Assuming mid-grey 50 as "unlit" is crude.
* `ShadowR_PythonRunner.cs` instead calls an external research model (repo `IgorAherne/Shadow_R`, downloaded as zip, run through a Windows `run.bat`). Not reusable for Overcast (Windows-only, external dependency); but the **UX pattern** (one-click "reduce shadows" creating a new layer from the result, with a model/download panel) is fine.

## 7. Vertex colours as an extra channel

`F/Camera/VertexColors/VertexColors_UserCamera.cs` and `F/Camera/View/VertexColors_ViewSpace.shader` render vertex colours as an additional **context image** (sent to the generator), and the
UV-chunk starting colour can come from vertex colours (`Objects_Renderer_MGR.ToggleVertexColors`). Handy for **region/part ID control images** (flat colour per part). Overcast has ID maps already; the code comment notes a bug workaround ("default Dungeon door shows white vertex colours only after Build"), so guard against meshes without a colour stream.

## 8. What to take

| Item | Mark | Effort | Value |
|---|---|---|---|
| UDIM rules and tests (§2) | ALGO | S | medium (correctness) |
| Separate chunk-coverage texture at stable resolution (§1) | IDEA | S | high |
| Internal 3-unit normalisation or bbox-relative tolerances (§3) | IDEA | S | medium |
| Temporary higher resolution during generation/projection (§4) | IDEA | S | medium |
| "Original mesh bytes stay, textures change" export contract (§4) | IDEA | S | medium (Blender bridge) |
| Flatten/merge layers op with confirmation (§4) | IDEA | S | medium |
| AO accumulation bake (§5) | ALGO | S-M | medium |
| Delight heuristics (§6) | IDEA | L | low |
