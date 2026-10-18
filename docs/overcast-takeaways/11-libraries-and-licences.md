# 11 - Third-party libraries, algorithms and their licences

Paths: `F/` = `Assets/_gm/Features/`. This is a factual inventory taken from the repository; licences of upstream projects that are only *named* (not vendored with a licence file) were **not verified online** (no network use was allowed) and are marked "unverified". Not legal advice.

## 1. Vendored code with licence files in the repo

| Component | Location | Licence (evidence) | Used for | Relevance to Overcast |
|---|---|---|---|---|
| **AssimpNetter** (fork of AssimpNet by Nicholas Woodfield, maintained by Stefan Koch; wraps Assimp) | `Assets/Plugins/AssimpNetter/` incl. native `assimp.dll`, `libassimp.so`, `libassimp.dylib` | MIT (`License.txt`: "Copyright (c) 2012-2020 AssimpNet - Nicholas Woodfield, (c) 2024-2025 AssimpNetter - Stefan Koch") | runtime import of OBJ/FBX/GLB etc., post-processing | Python equivalent: `trimesh`/`pyassimp`/`assimp` bindings (Assimp itself is BSD-3-Clause, unverified here) |
| **SimpleFileBrowser** (Süleyman Yasir Kula) | `Assets/Plugins/SimpleFileBrowser/` | MIT (`LICENSE.txt`) | in-app file dialogs | none (Tauri has native dialogs) |
| **UnityWindowsFileDrag-Drop** (Markus Göbel, Bunny83) | `Assets/UnityWindowsFileDrag-Drop/` | MIT (`LICENSE`) | Windows drag-and-drop of files into the app | none (Tauri handles it) |
| **UnityAI** (references SciSharp *Bigtree.Algorithm*) | `Assets/UnityAI-master/` | Apache-2.0 (`LICENSE`) | NLP support for prompt features | none |
| **UnityNLP** (references *OpenNLP*, *CherubNLP*; interface to WordNet) with model files (`EnglishPOS.nbin`, `EnglishTok.nbin`, `Parser`, `Coref`, `NameFind`) and `StreamingAssets/WordNet/dict` | `Assets/UnityNLP-master/`, `Assets/StreamingAssets/{Models,WordNet}` | Apache-2.0 (`LICENSE`); WordNet data is under Princeton's WordNet licence (unverified here) | highlight parts of the prompt text (`Settings_MGR.get_prompt_textHighlight`) | none |
| **UnityUtil** (`namespace Voxell`; radix sort and prefix-scan compute shaders `RadixSort.compute`, `Scan/*.compute`, `MeshUtil.cs`) | `Assets/UnityUtil-main/` | **No licence file in the folder**; upstream licence unverified (same `Voxell.*` namespace family as the Apache-2.0 UnityNLP/UnityAI packages, which suggests but does not prove the same licence) | GPU sort/scan helpers (not used by the projection pipeline: grep shows no `RadixSort`/`BlellochSumScan` call outside the folder) | none. Flag: unlicensed vendored code inside an AGPL repo is an upstream compliance question, not ours. |
| **ScrollView focus helper** (yasirkula gist, Feb 2024) | `C/UI (reusable)/Widgets and Gadgets/ScrollRect/ScrollRect_ItemFocuser_UI.cs:23-24` | MIT (comment) | UI scrolling | none |

Unity packages (`Packages/manifest.json`): `com.unity.formats.fbx` 5.1.4 (Autodesk FBX SDK wrapper, **proprietary SDK**; used by `ModelsHandler_SaveFBX_Helper.cs` for FBX export), `com.unity.nuget.newtonsoft-json` 3.2.1 (Newtonsoft.Json is MIT), `com.unity.burst`, `com.unity.mathematics`, `com.unity.inputsystem`, `com.unity.ugui`, TextMesh Pro, plus modules. These are Unity-licensed and irrelevant to a Python port.

## 2. External tools the app downloads or talks to (not in the repo)

| Tool | Where referenced | Notes |
|---|---|---|
| **A1111 / Forge / ComfyUI** (HTTP API) | `F/StableDiffusion/**` | SD plumbing, ignore. Contains docs links to AUTOMATIC1111 discussions and `Mikubill/sd-webui-controlnet` API pages in comments. |
| **Trellis / Hunyuan3D WebUIs** | `F/3D Generate/**` | image-to-3D through an external server; not relevant to texturing. |
| **rembg** (`IgorAherne/rembg-stable-projectorz`) | `F/Repos/Rembg`, `F/Paint/BackgroundRemoval/Rembg_PythonRunner.cs` | background removal for the 2D background and 3D-generation inputs. |
| **Shadow_R** (`IgorAherne/Shadow_R`) | `F/Repos/ShadowR`, `F/TextureTools/Delight/ShadowR_PythonRunner.cs` | shadow-reduction model for delighting; Windows `.bat` launcher; downloads zips with mirror URLs. |
| **spz-mcp** (redDwarf03, MIT per README) | root `README.md`, `F/AgentBridge/README.md` | MCP server over the agent-bridge socket. |
| **DiffusionLight** (Phongthawee et al., arXiv 2312.09168) | `F/Skybox + Background/MakeHDR_FromTextures2D.compute` header | method for HDR from chrome balls; paper, code licence not checked. |

## 3. Algorithms and sources named in code comments

| Algorithm / source | Where | Licence / status |
|---|---|---|
| NVIDIA "Solid Wireframe" (single-pass barycentric-distance wireframe) | `F/Camera/View/PreviewWithWireframe*.shader` header | paper technique; implementation adapted from Unity's `SpatialMappingWireframe.shader` (MIT per its header: "Copyright (c) 2016 Unity Technologies. MIT license") |
| Shadow-caster depth pass | `ProjectionVisibility_R8G8.shader`, `MultiProjectionShader.shader`, `Depth_MultiPOV_ShadowcasterReplacement.shader` | credited to `przemyslawzaworski/Unity3D-CG-programming` `shadowcaster.shader`; licence **unverified** |
| Unity mobile VertexLit fragment trick | same files | from `TwoTailsGames/Unity-Built-in-Shaders` mirror of Unity's built-in shaders; Unity Companion-style licence, **unverified** |
| HSV helpers `hue2rgb/hsv2rgb/rgb2hsv` | `C/Shader_Includes/ShaderEffects.cginc` | credited to Ronja's tutorials ("from https://www.ronja-tutorials.com/post/041-hsv-colorspace/", licence unverified) |
| SSAO with hemisphere kernel and noise rotation (16 samples) | `F/TextureTools/AO/SSAO_Bake.shader` | standard technique; sample kernel is a fixed table |
| CIE-LAB / sRGB↔XYZ conversion | `PatternAwareDelighter.compute` | standard constants (D65: 0.95047, 1.0, 1.08883) |
| Angular fisheye mirror-ball mapping and exposure merge | `SkyboxReconstruction.compute`, `MakeHDR_FromTextures2D.compute` | per DiffusionLight; math is standard |
| Hash noise `frac(sin(dot(uv,(12.9898,78.233)))·43758.5453)` and the 3-component hash (0.1031, 0.1030, 0.0973) | `EdgeDetect_ByDepth.compute`, `Improve_DepthContrast.compute` | widely used public-domain style hashes (the second is Dave Hoskins' "hash without sine") |
| Hierarchical min/max reduction; Hillis-Steele / Blelloch scans | `ReduceFind_MinMaxDepth.compute`, `UnityUtil` | textbook algorithms |
| `ShiftViewportCenter` / `PerspectiveOffCenter` matrix | `F/Camera/Logic/CameraTools.cs` | taken from Unity's scripting-reference example ("taken from https://docs.unity3d.com/ScriptReference/Camera-projectionMatrix.html"), the standard OpenGL off-axis frustum |

## 4. Libraries and algorithms useful for the Overcast stack (suggestions, not in SPZ)

| Need | Candidate | Licence (to verify before adoption) |
|---|---|---|
| Nearest-fill padding | OpenCV `distanceTransformWithLabels`, `inpaint` | Apache-2.0 (OpenCV 4.5+) |
| Depth-guided blur | `opencv-contrib` `ximgproc.jointBilateralFilter`, guided filter | Apache-2.0 / BSD |
| Mesh import in Python | `trimesh` (MIT), `pyassimp`, `open3d` | MIT / BSD |
| UV unwrap (no SPZ equivalent) | `xatlas` (MIT), Blender `bpy` smart UV | MIT / GPL |
| Seam-free colour blending | Poisson blending (`cv2.seamlessClone`), multi-band blending | Apache-2.0 |

## 5. Take-aways

* Everything *valuable* in SPZ for Overcast is original SPZ work (AGPL-3.0); the third-party pieces are plumbing (importer, dialogs, NLP) that Overcast does not need.
* If any SPZ shader text were ever copied, the embedded snippets from `przemyslawzaworski` and Unity built-ins would need separate licence checks. Another reason to describe-and-reimplement.
* The `UnityUtil` folder without a licence file should be treated as "all rights reserved" if anyone considers reusing it (nobody should need to).
