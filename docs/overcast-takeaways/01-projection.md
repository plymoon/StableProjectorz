# 01 - Camera projection, occlusion and visibility

Paths: `F/` = `Assets/_gm/Features/`, `C/` = `Assets/_gm/_Core/`. Reuse marks: `IDEA` / `ALGO` / `COPY?` (see README §1).

## 1. How SPZ projects an image onto UV texels

SPZ never ray-casts. Every projection is a **render pass into the UV atlas** in which the mesh is drawn with
`clip position = UV` (so the rasteriser walks the *texture* triangles) while the **camera-projected position travels along as an
interpolated attribute**. Each UV fragment therefore knows its screen position and just samples the screen-space art there.

Core files:

| Role | File |
|---|---|
| Single-view projection pass | `F/Camera/Projections/Shaders/ProjectionShader.shader` |
| Multi-view (up to 6) projection pass | `F/Camera/Projections/Shaders/MultiProjectionShader.shader` |
| Visibility (occlusion) texture per view | `F/Camera/Projections/Shaders/ProjectionVisibility_R8G8.shader` |
| Per-texel view alignment | `F/Camera/Projections/Shaders/Projections_Alignment.shader`, `Alignment_to_Visibility.compute` |
| Orchestration | `F/Camera/Projections/ProjCam_HelpTextures_Init.cs`, `ProjectorCamera.cs`, `ProjectorCameras_RenderHelper.cs`, `ProjectorCameras_MGR.cs` |
| Off-centre projection matrix | `F/Camera/Logic/CameraTools.cs` (`ShiftViewportCenter_ofProjMat`, `PerspectiveOffCenter`, `Set_POV_properties_into_mat`) |
| Per-frame driver | `F/Render/Objects_Renderer_MGR.cs` (`OnUpdate`, `cycle_through_generations`, `RenderProjection`) |

### 1.1 The texture-space trick (what the shader really does)

`ProjectionShader.shader`, vertex/geometry/fragment:

1. Vertex stage passes object-space position and UV through.
2. Geometry stage: computes the **flat face normal from positions** (`cross(e1,e2)`), compares with the averaged vector to the camera
   (`dot(normal, viewDir) >= 0`), and **drops back-facing triangles entirely**. Comment in the code: vertex normals are not used because
   smoothing groups can lie on low-poly meshes, and users may keep a clone rotated 180° (the back of the clone faces the camera and
   must not receive paint). It also selects the UDIM slice (`uv_to_renderTargIX`) and emits the triangle with
   `SV_Position = (uv*2-1, 0, 1)` (Y flipped) and, as a plain interpolant, `ComputeScreenPos(UnityObjectToClipPos(vertex))`.
3. Fragment: divides the interpolated homogeneous clip coordinate by its `w`, rejects if outside the unit screen square or behind the camera
   (`isOutsideScreen_or_behind` in `C/Shader_Includes/ShaderEffects.cginc`), samples art and masks.

**Why it is exact:** interpolating clip-space `xyzw` linearly with barycentrics that live in UV space, then dividing by `w` per texel,
reproduces the true perspective projection of the 3D surface point. (The 3D position is affine in the barycentrics, and projection is linear in homogeneous
coordinates.) The pitfall to avoid in a numpy port: interpolate **homogeneous clip coordinates** (or 3D positions), not NDC.

`ALGO` - numpy-style description for Overcast (own formulation, not a port):

```
# cached per mesh/UV layout:  tri_id[H,W], bary[H,W,3]  ->  P[H,W,3], face_n[H,W,3]
# per view v with view-projection VP_v and camera position c_v:
clip  = [P,1] @ VP_v.T                          # (H,W,4)
ndc   = clip[..., :3] / clip[..., 3:4]
uvs   = ndc[..., :2]*0.5 + 0.5                  # screen coords of every texel
inside= (clip_w > 0) & (|ndc.xy| <= 1)          # SPZ also rejects ndc.z behind the camera
facing= dot(face_n, c_v - P) >= 0               # flat face normal, never vertex normals
visible = inside & facing & (eye_depth(P) - bias <= depth_map_v(uvs))
```

### 1.2 Occlusion: `ProjectionVisibility_R8G8.shader`

Output per view and per UDIM slice, 8-bit two-channel texture in UV space:

* **G = "real" visibility**: 1 if the texel is on a front-facing triangle, inside the screen, not occluded, and not masked-out by the generator's
  screen mask; else 0.
* **R = "faded" visibility** = `G × edgeFade`, where `edgeFade` combines (a) a screen-space map of depth discontinuities and (b) a grazing-angle ramp.

Steps, in order (all in the one shader file plus the helper `ProjCam_HelpTextures_Init.Make_Visibilities_and_Alignments`):

1. **Depth pre-pass**: pass 1 of the same shader ("ShadowCaster", `ColorMask 0`, `Cull Back`, `ZWrite On`) renders the mesh from the projector POV
   into the camera depth buffer. Depth is read as `_LastCameraDepthTexture`.
2. **UV pass** (pass 2) rasterises into UV space (same geometry-shader culling as above) and in the fragment:
   * `if max(|ndc.xy - 0.5|) > 0.5  return 0`, i.e. outside the screen.
   * **Depth test with a constant bias**: `Linear01Depth(z) - 0.00003 > Linear01Depth(sceneDepth)` → occluded → `(0,0)`.
     The code comment explains that `0.001` leaked paint through a leather belt; `3e-5` was tuned for near `0.25`..tight, far `1000`, on meshes
     that are always rescaled into a **3-unit box**. In eye units that is about `3e-5 × far(1000) = 0.03`, roughly **1 % of model size**.
     The comment also warns that *pulling the far plane closer worsens quality* (non-linear depth) and that the depth buffer is only good if a shadow-casting
     directional light exists in the scene (an engine quirk we do not share).
   * If the generator's screen mask is 0 here → `(0,0)`.
   * `darkEdges = saturate(1 - blurredDepthEdges(screenUV))`.
   * Grazing-angle ramp with `n = normalize(vertex normals interpolated)` and `v = normalize(camera - pos)`:
     `ndv = saturate(dot(n,v))`; `ramp = 1 - saturate( inverseLerp(T, -2e-6, ndv) * 2 )` with `T = 0.3` when edge blur is on, `0` when off.
     Evaluating this: **ramp = 1 for ndv ≥ 0.3, falling linearly to 0 at ndv = 0.15** (≈72.5° to 81.4° from the normal); with `T = 0` there is no fade.
   * Output `(darkEdges × ramp, 1)`.
3. **Dilate 2 texels** (`DilateVisibility`, rule `NineSamplesAveraged`, keyed on channel G) to close the 1-texel gaps rasterisation leaves along triangle edges.
   (Missed texels would otherwise read as visibility 0 and later as bright seams.)

The blurred depth-edge map (step "darkEdges") is produced in `PrepareDepth`:

* `Depth_Contrast_Helper.Get_BlackWhiteDepth_POVs` renders linear depth (close = white, background = 0), then
  `Improve_DepthContrast(ApplyExact, 3 iterations)` normalises it (see `07`).
* `ScreenDepth_EdgesDetector.DetectEdges_ByDepth` (`F/TextureTools/EdgeDetect/EdgeDetect_ByDepth.compute`): for every non-background pixel, the maximum
  absolute depth difference over a 3×3 neighbourhood (a background neighbour counts as difference 1) is compared to a threshold
  `lerp(0.005, 0.1, edgesThresh01)`; the result is clamped to at least `1/255` so that the later blur can distinguish "inside silhouette, no edge" from "empty background".
* `BlurEdges_ofDepth`: 7 successive box-blur dispatches with a stride that grows with "age" (tight first, wide later) and a pseudo-random per-pixel jitter
  for the noisy mode; stride is multiplied by `max(w,h)/1024` to be resolution independent. This yields a soft rim: texels near silhouette edges fade out instead of
  producing hard stretched borders.
* For **multi-view** projections the edge map is skipped (cleared to 0): fading is delegated to the alignment weights (see `02`).

`ALGO` for Overcast: keep the **two-channel idea** (faded for blending, real for gating). Replace the constant `3e-5` by an eye-space bias of about 0.5-1 % of model size
**plus a slope-scaled term** (e.g. `k·tan(acos(n·v))`), which SPZ does not have and which avoids acne on grazing planes. (That addition is our suggestion, not in SPZ.)

### 1.3 The projection pass itself

Single view (`ProjectionShader.shader`, fragment):

```
art   = EffectsPostProcess(tex(screenArt, ndc.xy), HSV+contrast) * tint.rgb
mask  = (screenMask > 0) ? 1 : 0                    # hard threshold, see note
vis   = ProjVisibility[uv, slice]  (R faded, G real)
fine  = uvBrushMask[uv, slice].r * 2                # [0,1] -> [0,2]
w     = mask * opacity * saturate(fine - (1 - vis.R)) * vis.G
out   = float4(art.rgb, w)    with Blend SrcAlpha OneMinusSrcAlpha, One One
```

* The blend state is the key: **RGB = lerp(dst, art, w)**, **alpha = dst.a + w** (additive, saturating at 1 in UNORM). So layers stack in UI order, later on top.
* **Hard screen mask**: the comment says soft-inpaint masks used to cause "double fading" (generator already blended partial-denoise regions), so any mask value > 0 becomes 1.
  Worth copying as a rule: *when the generator was told to change region M, trust its output inside M fully and leave everything outside untouched.*
* **Fine-tune mask range [0,2]**: default stored value is 0.5 (gray). `fine = 2·M` makes the default exactly 1, so `saturate(fine - invisibility)` equals the faded visibility.
  A brush stroke can raise `M` up to 1.0 (`fine = 2`), which cancels fade (`invisibility ≤ 1`) and forces the layer to show at faded edges; erasing lowers it to 0 and hides it.
  See `04`/`05` for how the brush writes it.

Multi view (`MultiProjectionShader.shader`):

* The geometry stage computes **all six** `ComputeScreenPos` values (one `_ViewProj_matrixN` each) and emits **one** triangle, never one per view
  ("otherwise triangles would fight over the destination fragment and we wouldn't achieve a sequential blend"). Back faces are *not* culled here because the "wrong side" overlay must still be drawn.
* Fragment accumulates over POV `i` (all POVs read the **same** screen-art texture at their own screen coordinate; this is how a sheet works):

```
w_i       = saturate(2*M_i - (1 - Vfaded_i)) * Vreal_i
color    += w_i * art(ndc_i.xy)                 # note: no opacity, no screen mask in the colour term
weight   += opacity * screenMask_i * w_i
sumW     += w_i
out.rgb   = color / max(sumW, 1e-6)            # colour is normalised by weights, not by opacity (avoids dark borders)
out.a     = weight
```

  The comment in the code: not multiplying colour by `(artOpacity*screenMask)` is deliberate, otherwise "color becomes black around borders of projection, which looks ugly".
* `NUM_POV_2..6` are `multi_compile` keywords that make all variables for N views exist; `CameraTools.Toggle_numPOVs_Keywords` sets one keyword and the include
  (`C/Shader_Includes/MultiProjection_AutoDefines.cginc`) defines the lower ones. Not relevant to numpy, but it shows the **hard cap of 6 views per generation**.

## 2. Per-view geometry: principal-point shifting ("perspective shift")

`CameraTools.ShiftViewportCenter_ofProjMat(cam, cxy)` builds an asymmetric frustum from the camera's FOV and aspect:

```
h = 2·near·tan(fov/2);   w = h·aspect
p = (1,1) - c                                  # c = desired principal point in [0,1]^2, (0.5,0.5) = centred
left   = (p.x - 1)·w      right = p.x·w
bottom = (p.y - 1)·h      top   = p.y·h
P = PerspectiveOffCenter(left, right, bottom, top, near, far)     # standard OpenGL-style matrix
```

Result: the optical axis hits the image at `(c.x, c.y)` (Unity's bottom-up Y). Check: axis point `x=0` gives `ndc.x = 2·c.x - 1`. `ALGO`; add a unit test that
projects the look-at point and asserts it lands on `c`.

Use in SPZ: each of up to 6 view cameras gets its own `(pos, rot, fov, c)`; a `CameraPovInfo` stores exactly these (`F/StableDiffusion/GenData/GenData2D_Archive.cs:150`).
When rendering the **content image** that is sent to the generator, `UserCameras_MGR.RenderContentCameras` renders every active view into the *same* target (clear flags only for the first),
so the views appear side by side in one sheet (`F/Camera/UserCameras_MGR.cs:308-335`). After generation, the projection uses the same POV list, so each view reads its own region.
**This is the cleanest answer to cross-view palette drift in the repo**: one sampling run, one palette, N views.

`IDEA` for Overcast's context sheet: instead of cutting the sheet into tiles and projecting each tile with its own symmetric camera, represent every tile as
`(fov, c)` relative to the *sheet* image and project the whole generated sheet through all of them. Seams between views are then handled by the weights in `02`, and
palette consistency comes from the generator seeing all views at once (Qwen-Image handles a wide 2048 px canvas).

Extra: `FOV-compensated adjustment` (`UserCameras_MGR.StartFOV_compensatedAdjustment`, `CameraTools.Calc_PosOffset_forFOVchange`) moves the camera back/forward when FOV
changes so the subject keeps its screen size (dolly-zoom compensation: `d2 = d1·tan(fov1/2)/tan(fov2/2)`). Useful for a view-planner UI that lets users trade perspective for coverage.

## 3. What to take, effort, risks

| Item | Take | Effort | Risk / note |
|---|---|---|---|
| Hard back-face rule from flat face normals | Do exactly this; never vertex normals | S | none |
| Two-channel visibility (R faded / G real) | Yes, store both per view | S-M | storage ×2; 8-bit is enough but see `12` (banding in masks) |
| Depth-bias scaling relative to model size | Yes, plus slope-scaled term | S | tune on thin parts (belts, cloth) |
| Dilate visibility by 2 texels after UV raster | Yes (our rasteriser may already cover the gaps) | S | prevents bright seams |
| Hard thresholding of the generator mask | Yes | S | must match how Overcast builds masks |
| Off-centre projection (principal point) per view | Yes, central to the sheet | S (matrix), M (sheet pipeline) | only exact if the sheet image is exactly the render target |
| 6-view cap | Do not copy; arbitrary GPU-variant limit | - | our CPU/numpy path has no such cap |
| Geometry-shader approach, `SV_RenderTargetArrayIndex`, multi_compile variants | Do not copy; engine-specific | - | in moderngl use one FBO per UDIM or `gl_Layer` if needed |

Licence note: the shaders are original SPZ code under AGPL-3.0 (see README §1); the formulas and parameter values above are described, not quoted. The shadow-caster pass
cites `przemyslawzaworski/Unity3D-CG-programming` (licence not verified), another reason not to copy those blocks.
