# 10 - Performance tricks and how they translate to Python / numpy / OpenCV

Paths: `F/` = `Assets/_gm/Features/`, `C/` = `Assets/_gm/_Core/`. Reuse marks as in README §1.

SPZ is a real-time GPU application (Unity, DirectX/Vulkan). Overcast's core is a headless Python process with a three.js UI, so the *mechanisms* mostly do not transfer, but the *strategies* do.

## 1. Strategies that transfer

| # | SPZ strategy | Where | Overcast translation |
|---|---|---|---|
| 1 | **Dirty-flag rendering.** The whole UV composite is rebuilt only if `_renderAll_ASAP` is set (any slider/brush/layer event calls `ReRenderAll_soon()`) or the user is in multi-view edit mode (brush preview needs every frame). Otherwise only the display material is refreshed. | `F/Render/Objects_Renderer_MGR.cs OnUpdate` | Same: composite on demand with a version counter; return cached tiles. |
| 2 | **Expensive per-view textures are cached per generation**, and rebuilt only when a blend slider changes (`OnIcon_Some_Blending_Slider`), not per frame: visibility, alignment, depth edges. | `ProjectorCamera.Init`, `Render_the_Visibilities` | Cache `vis_faded`, `vis_real`, `weights` per (view, mesh version, params hash). |
| 3 | **Compute only what is needed.** `UserCameras_Permissions` keeps four lock sets (content, depth, normals, vertex colours): a camera renders only while someone holds a lock on its output (e.g., an enabled ControlNet that needs depth). Locks are held by object identity and released on dispose. Forcing all on is explicit (`Force_KeepRenderingCameras`) for saves. | `F/Camera/Logic/UserCameras_Permissions.cs` | Lazy context maps with reference-counted demand. |
| 4 | **Low-resolution working buffers for fuzzy operations.** Inpaint masks are processed at ≤512 px then resized; the AO preview is 1/4 resolution; the depth-of-mask search texture is 4× smaller. | `Inpaint_ScreenMasker_*`, `AmbientOcclusion_Baker`, `PreventBlur_if_DistanceFromMask` | Do blurs/dilations on 512 px and resize with `cv2.resize(INTER_LINEAR)`. |
| 5 | **Resolution management around the generator.** Texture resolution is cut to 256 while a generation runs (the generator shares the GPU), restored afterwards; raised to ~5k for the moment of inpaint capture; kept at 4k during project save. | `F/Settings/Performance_MGR.cs`, `SceneResolution_MGR.cs` | Same idea for the 5090 shared by Qwen-Image and the renderer: render the viewport at low resolution while sampling. |
| 6 | **Frame-rate policy**: target 70 fps, 2 fps when unfocused, optional throttle while generating, temporary un-throttle after file drops ("smooth fps after drag-and-drop"). | `F/Settings/Settings_MGR.cs` | UI side: pause render loops when hidden. |
| 7 | **Reduction and blur in few passes.** Min/max of a whole texture by a 4×4-per-thread pyramid; blur with "exaggerated far steps" `lerp(1, boxHalf, farSteps)` for a large cheap kernel instead of a huge one; blur iteration count scales with resolution (1 at <512, 2 at ≥512, 3 at ≥4096). | `Depth_Contrast_Helper.Blur_Depth_maybe` (iteration counts), `BlurTheTexture.shader` / `BlurInsideUVchunks.shader`, `BlurTextures_MGR.calc_StepAmplification` | `cv2.minMaxLoc`; `cv2.GaussianBlur`/`cv2.boxFilter` with sigma proportional to resolution. |
| 8 | **Random/accumulating algorithms are progressive** (AO bakes ~1024 iterations × 2 views; user can interrupt after a deliberate 0.25 s pause "for artistic look"). | `AmbientOcclusion_Baker.BakeAO_crtn` | Stream progress, allow early stop and use the partial result. |
| 9 | **Avoid allocations in loops**: `blitHelperTex` can be passed in so a dilation loop reuses one helper; `RenderTexture.GetTemporary`; intermediate pyramid textures kept in a list and reused when sizes match. README rule: "if you spawned a RenderTexture, release it". | `TextureDilation_MGR`, `Depth_Contrast_Helper.GetIntermediateRT` | Preallocate numpy buffers per tile size. |
| 10 | **8-bit and 16-bit formats for masks and helper data** (R8, R8G8, R16) to halve bandwidth. | `GenData_Masks`, `HelperTextures` | numpy `uint8`/`float16` where precision allows; but see `12` for 8-bit mask pitfalls. |
| 11 | **Async GPU read-back** for screenshots and eyedropper, never `ReadPixels` in the frame. | `TextureTools_SPZ.RenderTexture_to_Texture2D_Async`, `BrushRibbon_UI_EyeDropperTool` | Return tiles as binary frames over the websocket/HTTP in chunks. |
| 12 | **Parallel but bounded CPU work**: UDIM scanning runs `Task`s with a cap of 8 meshes in flight (to avoid memory blow-up). | `UDIMs_Helper` | `concurrent.futures` with bounded workers. |
| 13 | **Strict update order** (`Update_callbacks_MGR`): navigation → camera params → viewport depth → picking → brushing → UI → (late) content depth → object render → user render → screen masks. A stage can rely on earlier ones, avoiding one-frame lag artefacts. | `C/Logic/Callbacks + Events/Update_callbacks_MGR.cs` | Define the same dependency order in the render scheduler. |

## 2. GPU-specific tricks, and their CPU/numpy equivalents

| SPZ trick | Why it exists | Equivalent in Overcast |
|---|---|---|
| **Texture arrays, one slice per UDIM** + geometry shader that emits into `SV_RenderTargetArrayIndex` | one draw call for all UDIM tiles | A list of per-tile arrays, or one `(T,H,W,C)` array; vectorise over tiles. |
| `multi_compile` keyword variants (`NUM_SLICES_UPTO_8/16/24`, `NUM_POV_2…6`) | Unity cannot size arrays dynamically in shaders | Not needed. Do not import the 6-view / 24-tile caps. |
| Conservative rasterisation for the brush | guarantee coverage of thin UV triangles | Dilate triangle coverage by one texel, or sample by inverse mapping. |
| Compute groups of `(32,1,1)` threads, 4×4 texels per thread for reductions | occupancy tuning ("32 is good for NVIDIA, 64 for AMD") | numpy vectorisation; if porting to GPU (moderngl compute), use `local_size_x=32..64`. |
| Ping-pong full-texture blits for dilation | no scatter | `cv2.distanceTransformWithLabels` + gather (S effort, 10-100× faster for large radii). |
| Neighbour search with early-out in shaders (e.g., `PreventBlur` ±16 box × 16 line samples = ~17k reads per pixel) | simple GPU kernel | `cv2.ximgproc.jointBilateralFilter` or a distance-transform restricted by depth bands; cost matters at 512 px only. |
| `Linear01Depth` and near/far juggling | Unity depth buffers are non-linear and platform dependent | Use linear eye depth from the rasteriser. |

## 3. Costs worth knowing (from the code, not measured)

* Re-rendering the accumulation texture touches every layer's full-texture pass (no dirty rectangles): fine at 2048 on a GPU, a trap on CPU. For numpy, track **dirty tiles / bounding boxes** per layer and per stroke and recomposite only those.
* The brush applies a **full-texture compute** per frame (`ApplyBrushStroke_IntoMask` dispatches over all texels/slices). On CPU restrict to the stroke's UV bounding box (derived from the screen footprint through the UV→screen map, `05` §3.2).
* Dilation for export (`max(w,h)/16` iterations on 2048² = 128 passes ×2 blits) is run synchronously ("isRunInstantly: blocks the main thread"): a visible hitch. Use distance transforms off the UI thread.
* The visibility edge map runs a 3×3 edge search plus **7 blur dispatches** each time a slider moves; the depth-aware mask limiter does ±16 neighbourhood searches per pixel (at 512 px ≈ 262k pixels × 1089 candidates). Both are sized for 512 px working images; do not scale up the resolution without changing the algorithm.
* AO baking: 2 render passes × ~1024 iterations of a full UV-space SSAO with 16 samples per texel; seconds on a GPU, minutes on a CPU, so use a GPU path (moderngl) if AO is adopted.

## 4. Recommended order of adoption

1. Dirty-tile compositing and cached per-view textures (S-M, foundation).
2. Working-resolution policy (512 for masks; low-res viewport while generating) (S).
3. Distance-transform padding instead of iterative dilation (S).
4. Demand-driven context maps with reference counts (S).
5. Progressive bakes with cancel (S).
