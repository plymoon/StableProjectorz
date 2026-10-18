# 08 - Project file format and asset handling

Paths: `F/` = `Assets/_gm/Features/`, `C/` = `Assets/_gm/_Core/`. Reuse marks as in README §1.

## 1. Layout on disk

```
MyProject.spz                 JSON (Newtonsoft, indented), written with TypeNameHandling.Auto
MyProject_Data/               created next to the .spz; wiped and rewritten on every save (see pitfalls)
    FINAL_COMPOSITE_4K.png    composite of all layers, "in case the spz file gets corrupted, at least the user has the png"
    <model file bytes>        the originally imported mesh file (kept so the project is self-contained)
    <uv_mask_*.png, uv_visibil_*.png>   per view, per UDIM slice: `_uv_mask_{i}_{guid}.png` / `_uv_visibil_{i}_{guid}.png`
    <result images, context images>     generated textures and request byproducts (view, depth, normals, vertex colours, screen masks)
```

Code: `F/Save Load Import Export/ProjectSaveLoad_Helper.cs` (`SaveProj_crtn`, `LoadProject`, `CreateDataDir`, `Serialize_SPZ_toJSON`, `Save/Load_*_DataFolder`), `SerializationObjects.cs` (`StableProjectorz_SL` and friends, "SL" = save/load DTOs), `Save_MGR.cs` (menu actions).

* A **DTO per subsystem** (`ProjectorCameras_SL`, `Generations2D_MGR_SL`, `ModelsHandler_3D_SL`, `UserCameras_MGR_SL`, `BrushRibbon_UI_SL`, …) is gathered into one root object (`StableProjectorz_SL`). Each manager has `Save(spz)` / `Load(spz)` and an `OnAfter_AllLoaded` phase for cross-references (e.g., projector cameras find their `GenData2D` by GUID after the archive loaded). The load order is hard-coded and commented ("ModelsHandler_3D.Load before projector cameras").
* **Images are PNG**, 8-bit; texture arrays are written one PNG per UDIM slice and listed with their `UDIM_Sector` (`RenderUdims_SL { textures, udim_coords }`). Masks therefore round-trip at 8-bit.
* Data folder path is **derived from the project path at load time** (`update_dataDir_toCurrent`), not trusted from the stored string, so projects survive being moved or renamed together with their `_Data` folder. (The root also stores `filepath_with_exten`/`filepath_dataDir` as strings, which are stale after a move; good that they are overridden.)
* The model is stored as **raw bytes of the imported file** (`ModelsHandler_3D.Get_3dModel_asBytes`, `_modelBytesCache`), and meshes are identified by stable `ushort unique_id` values stored in `SD_3D_Mesh_SL` plus their UDIM sectors.

## 2. What is stored per generation (provenance)

`GenData2D_SL` (`SerializationObjects.cs`) contains:

| Field | Purpose |
|---|---|
| `internal_GUID`, `kind` | identity and role (`02`/`04`) |
| `selected3dModel_pos` | where the selected model was, to restore cameras relative to it |
| `camerasPOVs : List<CameraPovInfo>` | pos, rot, fov, principal point, enabled flag for up to 6 views |
| `_projCamera_ix` | which projector camera object belongs to it |
| `byproductsOfRequest` | paths to the context images: `usualView`, `depth`, `viewNormals`, `vertexColors`, `screenSpaceMask_WE`, `screenSpaceMask_NE` |
| `txt2img_req` / `img2img_req` / `ext_req` | the full request payloads (SD-specific; Overcast would store Qwen prompt, seed, steps, sampler, model id, tile size) |
| `generatedTextures` | list of result images, GUIDs, UDIM sectors, which texture is preferred |
| `masking_utils` | per-view brush masks + visibilities (PNG lists) |
| `n_iter`, `batch_size` | batch information |

This makes every layer **re-loadable, re-projectable and re-runnable**: SPZ can restore the exact camera setup of a layer (`Restore_CamerasPlacements`), re-render its visibility, or regenerate with the same parameters, because the camera, context images, mask and request are one record. `IDEA`/value high for Overcast Stage 4 and 6: define a `Generation` record with exactly these fields (`camera_set`, `context_images{view, depth, normal, id, mask}`, `request`, `results[]`, `masks{per view}`), referenced by layers by id.

Other root fields: workflow ribbon state (denoise strength, mask blur step, mode), brush state (size, hardness index, opacity, colour), UI toggles, background gradient colours, scene resolution, ControlNet unit settings, connection panel (server URL), skybox colours. Mixing **document data and UI state in one file** is convenient but couples the format to the UI (see pitfalls).

## 3. Versioning, migration, safety

* `StableProjectorz_SL.version` is written (from `SP_Version.currVersion`) but **never checked or migrated** on load; incompatibility shows as a generic "file is corrupted, or an unsupported version" message when deserialisation throws.
* `TypeNameHandling.Auto` is enabled for both save and load (`StableProjectorz_SL.CreateFromJSON`, `Serialize_SPZ_toJSON`): a known deserialisation-attack surface for untrusted project files (Newtonsoft documents it as unsafe without a binder). Overcast must not copy this; use plain JSON with explicit discriminator fields and a schema version.
* **Saving wipes the data folder first**: `CreateDataDir` deletes every file and sub-folder inside `<name>_Data` before writing, then writes JSON, then textures, then the final composite. It is not atomic: a crash or full disk mid-save leaves a broken project, and any user file placed in that folder is lost. The only protection is the file-browser's overwrite prompt on the `.spz`.
* Saving is refused while generating ("Can't save while generating."), and the save path temporarily raises resolution to 4k and waits frames for the composite (`02`/`03`).
* Hotkeys: `Ctrl+S` save, `Ctrl+L` load (`Save_MGR.cs:349-350`; these use the left Ctrl only, and after file dialogs the code **resets the keyboard/mouse/pen input state** because Unity keeps modifier keys "stuck").

## 4. Asset handling

* Import images by drag-and-drop (Windows only, via the vendored `UnityWindowsFileDrag-Drop`), by file browser (`SimpleFileBrowser`), or via "load custom image into this projection" on a layer (`Images_ImportHelper.cs`). Multi-texture import maps textures to UDIM tiles by filename (`03`).
* Export set (`ExportSave_UI_MGR.cs`): final texture with dilation, without dilation, view textures (content/depth/normals/vertex colours as separate `_Content/_Depth/_Normals/_VertCols` PNGs), all art icons, all background icons, 3D model with textures, save/load project. A slide-out panel opens on hover.
* Background removal and delighting are delegated to downloaded Python/Windows tools placed beside the executable (`F/Repos/*`: zip downloads with mirror URLs, progress UI, `rembg`, `Shadow_R`). Not suitable as-is for Overcast (Windows `.bat`, unpinned downloads), but the **first-run "download optional component" panel** pattern is good UX.

## 5. Recommendations for the Overcast format

| Idea | Mark | Effort |
|---|---|---|
| One record per generation containing camera set + all context images + request + masks (§2) | IDEA | M |
| Derive data-folder location from the project path at load (§1) | IDEA | S |
| Embed the source mesh bytes or a content hash (self-contained project) | IDEA | S |
| Keep a flattened composite PNG next to the project for recovery (§1) | IDEA | S |
| Per-subsystem DTOs with `Save/Load/OnAfterLoaded` phases and a schema `version` + migrations | IDEA | M |
| Write to a temp folder then atomically rename; never wipe user content | IDEA | S |
| 16-bit PNG (or EXR/NPY) for masks and weights rather than 8-bit | IDEA | S |
| Do not mix UI-only state into the document file | IDEA | S |
