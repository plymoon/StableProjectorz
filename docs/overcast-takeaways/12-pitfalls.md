# 12 - Bugs, limitations and user-hostile design to avoid

Paths: `F/` = `Assets/_gm/Features/`, `C/` = `Assets/_gm/_Core/`. Everything below was found by reading code and comments; nothing was executed, so visual consequences are marked "likely" where inferred.

Legend: **[B]** bug or inconsistency in code; **[L]** hard limitation; **[U]** user-hostile behaviour; **[M]** maintenance trap; **[R]** risk for Overcast if copied.

## 1. Data safety and workflow

| # | Item | Evidence | Lesson for Overcast |
|---|---|---|---|
| 1 | **[U] No undo/redo** for strokes, fills, mask inversion, flatten, layer deletion, mesh removal or import. The UI warns "There is no CTRL+Z yet." in at least 8 confirmation texts (`F/Icons/IconUI_List_Art/IconsUI_List.cs:206,230,246`, `Art2D_IconsUI_List.cs:77`, `F/3D Models/UI/SD_subMesh_IconUI.cs:68`, `ModelsHandler_3D_UI.cs:43,121`). | no undo code under `F/Paint`, `F/Camera`, `F/StableDiffusion`; comment in `Inpaint_MaskPainter.ApplyColorLayer_To_UV_Textures`: "Never blit colors if saving. Or merging icons into one, because no ctrl+z." | Stroke-command undo with bounded patch history (`04` §3). |
| 2 | **[U] Layer reordering is disabled** ("NOT USED NOVEMBER 2024. Preventing movement of icons, because Save/Load gets broken."), so draw order is creation order. | `F/Icons/IconUI_List_Art/Art2D_IconsUI_List.cs ChangePriority_IconGroup_maybe` | Key layers by stable ids, store order separately. |
| 3 | **[U] Saving deletes everything inside `<project>_Data`** before writing, non-atomically; project save is refused during generation. | `ProjectSaveLoad_Helper.CreateDataDir` | Write to a temp dir and swap; never delete user files. |
| 4 | **[U] Changing "brush UV precision" in settings deletes all art icons** (a confirm popup warns, but all generations are disposed). | `F/Settings/Settings_MGR.cs set_brushPrecision_res → GenData2D_Archive.Dispose_ALL_genData()` | Make mask resolution a per-layer property, resample on change. |
| 5 | **[R] `TypeNameHandling.Auto` on project load** (deserialisation of arbitrary types from an untrusted file). | `SerializationObjects.cs CreateFromJSON`, `Serialize_SPZ_toJSON` | Never. Use explicit discriminators. |
| 6 | **[B] Version string written but never checked/migrated.** | `StableProjectorz_SL.version` | schema version + migration table. |
| 7 | **[U] Destructive/global operations hide behind hotkeys that depend on hover and modifiers** (`Ctrl/Shift+E` delete colour layer, `Ctrl/Shift+F` fill, `R` held switches the workflow mode to ProjectionsMasking "to avoid user frustration"). | `BrushRibbon_UI_DeleteButton.cs`, `BrushRibbon_UI_BucketFill.cs`, `MainViewport_UI.ShowOrderOfProjections_maybe` | Hotkeys must never change mode as a side effect; show confirmations/undo instead. |

## 2. Algorithmic and numerical

| # | Item | Evidence | Lesson |
|---|---|---|---|
| 8 | **[B/R] Likely colour darkening at partial alpha.** Layers blend with `Blend SrcAlpha OneMinusSrcAlpha, One One` into a transparent-black target: on empty texels RGB becomes `src·w` while alpha becomes `w`. No pre/un-premultiply handling exists (grep). Later padding averages that dimmed RGB outward. | `ProjectionShader.shader:24`, `02` §2 | Keep straight colour + separate coverage; premultiply only in the composite. Needs verification on a real run. |
| 9 | **[L] Hard caps**: 6 views per generation; 24 UDIM tiles; only the best 4 views per texel in alignment weights. | `MultiProjection_AutoDefines.cginc`, `UDIMs_Helper.MAX_NUM_UDIMS`, `Projections_Alignment.shader` | Do not inherit caps from shader-variant limits. |
| 10 | **[B] Triangle → UDIM slice from the first vertex only** (`uv_to_renderTargIX(g[0].uv)` / `IN[0].uv`). Triangles crossing a tile border land entirely in one tile. | `ProjectionShader.shader:138`, `ProjectionVisibility_R8G8.shader:193`, many others | Choose per triangle by centroid; split or clamp. |
| 11 | **[B] Dead parameter**: `_BestPov_DotMinRange` is set from `edgeBlurStride_01` but never read by the kernel. | `Alignment_to_Visibility.compute:36`, `ProjCam_HelpTextures_Init.cs:239` | Remove or implement; a slider that does nothing confuses users. |
| 12 | **[B] Downsample loop evaluates one texel instead of the commented 5×5**: `for(offset = BoxSizeHalf; offset <= BoxSizeHalf; …)`. | `Camera/Depth/Shaders/Downsample_depthOfMask.compute` | Harmless here but shows loops are not tested against comments. |
| 13 | **[M] Dead/unused code kept in the tree**: legacy bit-packed mask atlas kernels (zero references), `ApplyMild` depth kernel (no caller), `AntiShade` mode, commented-out reordering, debug fields tagged `//MODIF` (`HDR_PanoSkybox_MGR`, `Delight_MGR`). | see `04` §2.3, `07` §3.1 | Delete experiments or move to a branch. |
| 14 | **[R] Depth bias is a magic constant (`3e-5` of Linear01 depth)** tuned for near 0.25 / far 1000 and meshes normalised to a 3-unit box; the code warns the depth buffer is "severely degraded" unless a shadow-casting directional light exists in the scene (engine behaviour). | `ProjectionVisibility_R8G8.shader:222-233`, `Object3d_BrushShader.shader` | Scale-aware bias + slope term; compute depth ourselves. |
| 15 | **[R] Culling uses flat face normals from vertex winding**; correct only if triangle winding is consistent. The importer applies `FlipWindingOrder` to compensate for its coordinate conversion. Single-sided thin geometry (leaves, cloth) cannot receive paint from behind (they show a "WRONG SIDE" overlay instead). | `AssimpLoader.cs`, `MultiProjectionShader.shader` | Detect and report inconsistent winding on import; offer two-sided mode. |
| 16 | **[R] Padding can leak across thin gutters** (nearest-neighbour growth reaches the neighbouring island): the bucket-fill code limits padding to 1 texel for that reason. | `MaskPainter.OnBucketFill_orDelete_button` comment | Pad into gutters per island id or cap radius. |
| 17 | **[L] 8-bit masks and visibility** (`R8`, `R8G8`), with the mask value range halved by the [0,2] encoding; also stored as 8-bit PNG. Soft strokes may band; accumulation of many small edits quantises. | `GenData_Masks` formats | 16-bit masks. |
| 18 | **[M] No dirty rectangles**: full-array compute per stroke frame and full re-composite per change; memory grows with `#generations × #views × #UDIMs` (a single 2048² generation with 6 views holds about 6 × (1 + 2) bytes × 4.2 M texels ≈ 75 MB of mask/visibility per UDIM, 4× that at the 4096 option). | `GenData_Masks`, `ApplyBrushStroke_ToUvMask` | Tile and cache; store masks sparsely. |
| 19 | **[B] Inconsistent "same" thresholds scattered as literals**: 0.01 (dilation), 0.1 (border detect), 0.95 (coverage), 0.3/0.15 (angle ramp), 0.2/0.5 (brush normal fade), 0.15 units (mask blur depth limit); no shared constants. | various | Centralise tunables with names and units. |
| 19b | **[U] Overlapping or mirrored UVs silently break results** and are only mentioned in launch-screen hints ("Black 3d models? Ensure UVs are non-overlapping", "Symmetry? Arrange 2 halves into UDIMs"); no import-time validation. | `LoadIntroScreen_Hints.cs` | UV validation on import (`03`). |

## 3. Input, platform and architecture

| # | Item | Evidence | Lesson |
|---|---|---|---|
| 20 | **[M] Mixed legacy `Input.GetKey*` and the new Input System** in the same classes; after file dialogs the code **force-resets keyboard/mouse/pen state** because modifier keys stay "stuck". | `Save_MGR.ResetCtrlKey_AfterLoadSave`, many `Input.GetKeyDown` | One input layer; no stuck-key hacks (the React UI gets this for free). |
| 21 | **[U] Hotkey overload and implicit guards**: `Q` = hide layer / fly down; `A/D` = previous/next layer / fly left-right; `F` = focus / `Ctrl+F` fill; digits = opacity but `Shift+digits` = tabs; each class re-implements "is input field active / RMB pressed / hovering viewport". Save/Load check `LeftControl` only. | `Art2D_IconsUI_List.cs`, `CameraMove.cs`, `CommandRibbon_UI.cs`, `Save_MGR.cs:349` | Central hotkey registry with context rules and conflict detection. |
| 22 | **[M] `StopAllCoroutines()` as a synchronisation tool** (`Screenshot_MGR`, `ProjectSaveLoad_Helper.SaveProject`): concurrent operations cancel each other (their own agent-bridge README documents the consequence). | `F/AgentBridge/README.md` "Known limits" | Explicit task objects with cancellation tokens. |
| 23 | **[M] Singleton-everywhere architecture** (README rule "talk between scripts via `MyScript.instance`"), with "scenes are probably still loading" early-returns scattered across hundreds of methods. | `README.md` §Contribution Rules; many `if(instance==null){ return; }` | Dependency injection and an explicit lifecycle. |
| 24 | **[M] Copy-paste shader maintenance**: the same geometry-shader culling block appears in ≥ 8 shaders; `Object3d_BrushShader` has a full duplicate without conservative raster ("you must update too if changing this shader"); multi-variant keywords multiply compile time. | `Object3d_BrushShader*.shader` | One implementation, parameterised. |
| 25 | **[R] Windows-first**: `.bat` launchers for external tools, Windows-only drag-and-drop hook (disabled in the Editor), `%LOCALAPPDATA%` token path, `Application.dataPath` assumptions. | `ShadowR_PythonRunner.cs`, `UnityWindowsFileDrag-Drop/README.md` | Keep Overcast launchers cross-platform. |
| 26 | **[U] Blocking work on the main thread**: `isRunInstantly` dilation loops (up to 128 iterations at 2048) during export and in the per-frame anti-seam step. | `TextureDilation_MGR.dilate_crtn` | Background workers + progress events. |
| 27 | **[R] Tight coupling to Stable Diffusion**: payload structures, ControlNet unit panels, A1111 inpainting parameters live next to generic code (`GenData2D` holds `SD_txt2img_payload`, etc.). | `GenData2D_Archive.cs` | Keep the generation backend behind an interface; store requests as opaque JSON with a backend id. |
| 28 | **[U] Scale policy hides unit problems**: every mesh is silently rescaled so its largest side is 3 units (position recentred); exports re-apply `currModelRoot_scaleAfterImport`; a warning appears only for extents > 1001. | `Objs3D_Container.RescaleModel_fitIntoVolume` | If normalising, keep an explicit transform and show it. |
| 29 | **[B] Behavioural surprises documented only in comments**: soft vs hard masks ("double fading"), dilation radii changed by user requests (2 → 5 px, Discord), fixed 512 px mask size "probably works just for the current INPAINT_MASK_MAX_PX_SIZE" (the `/1.5` dilation factor). | `Objects_Renderer_MGR.AntiSeams_maybe`, `Inpaint_ScreenMasker_Original.CalculateBoxBlurDilation` | Turn tuned numbers into documented, tested parameters. |

## 4. Quality of life that SPZ gets wrong and we can do better

* Layer panel without names, groups, locks, or blend modes (`04`).
* No symmetry, flow, spacing, stabiliser in brushes; no clone/smudge/eraser on colour (`05`).
* Everything is re-rendered on any change; slider changes recompute depth edges and visibility synchronously (`10`).
* Camera/view state is partially global (a single `Cameras_MGR` with 6 slots); no named viewpoints or saved view sets besides the ones stored with generations (`07`).
* No automatic coverage planning, so a user must find uncovered regions by eye (`07` §5).
* Export dialogs always ask for a base file name, then silently add `_AO`, `_Content`, `_Depth`, … suffixes and UDIM numbers (`03`).
