# Materials, Rendering & VFX

Everything a Lens draws to the screen passes through this layer: **materials** decide how surfaces respond to light, **the render pipeline** (Cameras → Layers → Render Targets) decides what gets drawn and in what order, and **particles/VFX/post-effects** add motion and full-screen style. All of it runs on a mobile GPU inside a hard frame budget, so the recurring theme of this area is *visual richness vs. performance* — nearly every knob here trades one for the other. This file targets **Lens Studio 5.23.x mainline** (current build **5.23.1**, Aug 5 2026); the **Spectacles (2024)** target is pinned to **LS 5.15.4**, so validate any 5.2x rendering/VFX feature against the Spectacles docs before shipping to that device. Prefer the unversioned `developers.snap.com/lens-studio/...` doc paths — the `/4.55.1/...` copies are frozen legacy.

## What is possible

- Skin 2D/3D objects with GPU shaders using **five built-in base material types** or author **custom node-based shaders** in the Material Editor (Shader Graph).
- Control transparency and blending with **12 blend modes** and remove manual sort order via **Order-Independent Transparency (OIT)**.
- Build the render pipeline from **Cameras**, **Layers**, and chainable **Render Targets** for multi-pass effects.
- Apply full-screen **Post Effects** (color grading, blur, stylization) as graph materials.
- Simulate up to ~1M particles with the node-based **VFX Editor**, or use the cheaper deterministic **GPU Particles** for simple effects.
- Light and shadow 3D content with real-time **Shadow Mapping**, image-based lighting, and (on capable devices) **Ray Tracing** for reflections and global illumination.

## Key components, assets & APIs

**Base material types** — the five listed on the [Materials Overview](https://developers.snap.com/lens-studio/features/graphics/materials/overview):

- **Unlit** — flat, unshaded, texture-only. Cheapest; prefer for performance.
- **Diffuse** — basic shaded; uses scene lighting + env maps.
- **PBR** — physically-based (metallic/roughness). Most expensive of the standard set. (Note: **PBR is the current name**; older text called this "Uber" — do not present "Uber" as an official type.)
- **Occluder** — masks objects behind it by writing depth but not color.
- **Matte Shadow** — invisible surface that renders only *received* shadows, for grounding AR objects.

**Graph Materials are not a sixth base type.** They are custom shaders authored via Asset Browser **+ → Graph Unlit / Graph PBR** and edited in the Material Editor. Source: [Material Editor Introduction & Concepts](https://developers.snap.com/lens-studio/features/graphics/materials/material-editor/introduction-and-concepts).

**Common material properties** — Base Color + Base Texture, separate Opacity Texture, Metallic (0–1), Roughness (0–1), Material Params texture (packs metallic/smoothness/AO), Normal map, Emissive + intensity, Detail maps/masks, Rim highlight (Fresnel), Simple Reflection / Camera Reflections, Specular AO, Baked Shadow, Fizzle (noise dissolve), Tone Mapping (HDR). **Rendering controls:** Two-sided, Depth Test, Depth Write, Depth Function, Cull Mode, Polygon Offset, Color Mask, Frustum Culling.

**Blend modes (12, verbatim):** Disabled, Normal, Multiply, Add, Premultiplied Alpha, Glass, Colored Glass, Alpha Test, Alpha To Coverage, Screen, Min, Max. An Opacity Texture requires **Normal**; **Alpha Test** (cutout) is cheaper (no sorting).

**Render pipeline** — `Camera` (every Lens needs at least one; see the [Camera scripting class](https://developers.snap.com/lens-studio/api/lens-scripting/classes/Built-In.Camera.html)), **Layers** (gate what a Camera renders), **Render Target** (output surface, chainable for multi-pass), scene-config **Live Target** / **Capture Target**, Render Order + Mask Texture, plus Depth Render Target and Device Depth. Source: [Camera](https://developers.snap.com/lens-studio/lens-studio-workflow/scene-set-up/camera).

**Particles/VFX** — **VFX Asset** + **VFX Component** (mirrors the Material + RenderMeshVisual split); classic **GPU Particles** = a Mesh Visual with a **Particles Emitter Material**. **Post Effects** are full-screen **Graph Post Effect Materials**.

## How to build it

**Author a custom shader (Material Editor / Shader Graph).** Create a Graph material (**+ → Graph Unlit / Graph PBR**), then wire nodes left→right; the graph terminates at the **Shader** node and cannot contain feedback loops. Node categories: Math, Inputs/Parameters, Functions (Texture Sampling, Coordinate, Color Conversion, Noise, Utility), Loops, Sub-Graphs. For raw shader logic, drop a **Code Node** and write **"almost entirely native GLSL"** — a cross-compiler emits device-safe shaders. Declare ports as `input_float`, `output_vec4`, etc. Sources: [Nodes reference](https://developers.snap.com/lens-studio/references/material-editor/nodes), [Code Node guide](https://developers.snap.com/lens-studio/features/graphics/materials/material-editor/code-node/code-node-guide).

**Handle transparency correctly.** Transparent (`Normal`-blend) objects render **back-to-front**; by default you sort them manually in the hierarchy, which breaks on intersecting geometry. Enable **OIT** per-Camera via the **OIT Layers** option (applies only to Normal-blend materials): **No OIT** (free) → **4 Layers** → **4 Layers +1 Front** → **8 Layers** (slowest). Beyond the layer limit, excess layers render unsorted. Two-sided materials consume extra OIT layers; disable **Depth Write** on OIT objects. The editor's **Visualize Layer Count** overlay reads **green ≤4, yellow 4–8, red >8**. Source: [Order-Independent Transparency](https://developers.snap.com/lens-studio/features/graphics/advanced/order-independent-transparency).

**Add a full-screen Post Effect.** Scene Hierarchy **+ → Post Effect**; hierarchy placement = render order. Built-ins include Color Correction (LUT), Color Gradient, Color Remap, Distortion, Dithering, Edge Detection, Gaussian Blur, Halftone, Oil Paint, Pixelization, Shake, Smoothing, Zoom Blur, Analog TV, VHS. Each is a full-screen fragment pass — use sparingly. Source: [Post Effect](https://developers.snap.com/lens-studio/features/graphics/materials/post-effects).

**Build node-based particles (VFX Editor, since LS 4.0).** Non-deterministic (reads the prior frame for realism), scales to ~1M particles. Containers flow **Emit → Initialize → Update → Output** (Output renders as **Quad** default or **Mesh**, with Render Order Offset + Layer Override; multiple Output nodes allowed). Read/write attributes (Position/Velocity/Life/Force/Mass/Color/Size/Matrix/Custom are modifiable; Age/Delay/Seed/Index/Copy ID are system-set) via `Particle (Get/Modify Attribute)`; use the 50+ Container Sub-Graphs, Sort By Depth, and Warmup/Pre-Roll. Script parameters with `script.myVFX.properties.myCustomFloat = 0.9;`. **(LS 5.22)** enable **"Pause When Not Visible"** to freeze VFX when the object leaves the camera view, and use the consolidated **Particle Orient Node** (Billboard / Velocity Align / Look At). Sources: [VFX Overview](https://developers.snap.com/lens-studio/references/vfx-editor/overview), [ar.snap.com/lens-studio-v5](https://ar.snap.com/lens-studio-v5).

**Build classic particles (GPU Particles).** **+ → Particles** creates a Mesh Visual + Particles Emitter Material. Deterministic (ignores prior frame). Properties: Instance Count, Mesh Type (Quad vs 3D), spawn Box/Sphere, Instant/Constant Spawn, Pre-Warm, Flipbook, Color/Velocity Ramp, Velocity + Drag, Gravity, Local/Custom Force, Noise. **Trails added in LS 5.4.0.** Source: [GPU Particles Overview](https://developers.snap.com/lens-studio/features/graphics/particles/gpu-particles/overview).

**Light and shadow 3D content.** Light types: **Point, Directional, Ambient, Environment Map Light (IBL)**, plus **Spot** lights; a **Dynamic Environment Map** builds a real-time env map from Device Camera Input. Directional and Spot lights cast real-time shadows via **Shadow Mapping** (depth-based, self-shadowing, Soft Shadows, Shadow Quality, Shadow Bias); **Projective Shadows** are cheaper (no depth, no self-shadow). Directional/Spot Shadow Mapping was added/expanded in **LS 5.20**. Source: [Light and Shadow](https://developers.snap.com/lens-studio/features/graphics/light-and-shadow).

**Enable Ray Tracing (capable devices only).** Add a **Ray Tracing** asset and enable it per-Camera for **Reflections + Global Illumination**; use the **Ray Tracing Reflections** node in the Material Editor and a **Reflected Material** override to swap in simplified materials. **Caster meshes are limited to 65,000 triangles.** Supported devices: **iPhone 11–16, Samsung Galaxy S20–S25, Google Pixel 6–9, OnePlus 8–13.** Source: [Ray Tracing guide](https://developers.snap.com/lens-studio/features/graphics/raytracing/raytracing-guide).

## Best practices

- **Pick the cheapest material that looks right:** Unlit > Diffuse > PBR in cost. Reserve PBR for hero objects that genuinely need metallic/roughness response.
- **Prefer Alpha Test over Normal blending** for hard-edged cutouts — it skips sorting entirely, avoiding OIT cost and sort artifacts.
- **Reach for OIT only when manual hierarchy sorting fails** (intersecting/overlapping transparent geometry). It fixes sorting at real GPU cost, so use the lowest layer count that looks correct and watch the Visualize Layer Count overlay.
- **Choose the right particle system:** GPU Particles for simple, performance-critical, broadly-compatible effects; VFX Editor for complex, interactive, realistic simulation. Source: [VFX vs GPU Particles](https://developers.snap.com/lens-studio/features/graphics/particles/vfx-editor/vfx-vs-gpu-particles).
- **Respect texture budgets:** 3D textures are typically authored at **2048×2048** — reduce to 1024/512 where possible (the docs give 2048 as the common authoring size, not a hard cap); size screen-space textures to the space they occupy, max **720×1280** (Snapchat's Lens output resolution — per Snap's [official optimization tips](https://medium.com/lens-studio/tips-and-tricks-e69f1092d0f4), not the docs page). PNG for alpha, JPG for opaque. **Never auto-compress normal maps or environment maps** ("auto compression tools might result in bad normals"). Source: [Texture Optimization](https://developers.snap.com/lens-studio/publishing/optimization/texture-optimization).
- **Import meshes as FBX, glTF (.gltf/.glb), or OBJ** (OBJ carries meshes/textures only; FBX & glTF also carry animations/skeletons). Stay under **100,000 triangles** total, **60,000** skinned, **100 joints/rig**, animations < 10 s. Sources: [3D Import Overview](https://developers.snap.com/lens-studio/assets-pipeline/3d/importing-content/overview) (formats), [Performance Optimization Guide](https://developers.snap.com/lens-studio/publishing/optimization/performance-optimization-guide) (budgets).
- **Post Effects are full-screen passes — stack them sparingly**; each one re-samples the whole frame.
- **(LS 5.22)** Node-graph binaries are deprecated and auto-convert to **YAML** text formats (`.graphShader`, `.graphVfx`), which are diff-able and editable externally — useful for version control and AI-assisted editing.

## Hard performance budgets (Lens-wide)

Enforce these from the [Performance Optimization Guide](https://developers.snap.com/lens-studio/publishing/optimization/performance-optimization-guide):

- **Lens size ≤ 8 MB** at submission.
- **RAM:** the Performance guide says **≤ 150 MB**; the Texture Optimization page says **< 120 MB**. Both are current — treat **~120–150 MB** as the practical band and cite the page you rely on.
- **Lens Activation Time ≤ 650 ms** for sponsored/ad Lenses.
- **Target 30 FPS; never below 15 FPS.**
- **≤ 100,000 triangles** total / **≤ 60,000** skinned; **≤ 100 joints/rig**; animations < 10 s.
- **≤ 10 Liquifies per render order.**
- Lighting caps: max **3 directional + 5 point lights** active at once; **only one light casts shadows** at a time; only Directional/Spot cast real-time shadows.
- Minimize render targets; disable MSAA and unused Depth-Test/Depth-Write/Two-sided/alpha; prefer Unlit over PBR; tie logic to events; one ML Component per Lens.

## Common pitfalls

- **Intersecting transparent geometry flickers or sorts wrong** — manual hierarchy sorting can't resolve per-pixel intersections. Symptom: front/back faces pop as the camera moves. Fix with OIT (and disable Depth Write on those objects).
- **Assuming "PBR/Uber" or "Graph Material" is a base type.** Only five base types exist; Graph materials are custom-authored shaders, and "Uber" is legacy terminology.
- **Auto-compressed normal/environment maps look broken** — banding, wrong lighting. These must skip auto-compression on import.
- **Code Node undo/redo is unreliable** (~5 actions behind) — save often and don't trust rapid undo. In **VFX containers**, a container-attached Code Node **cannot set a custom particle attribute** and **output-port creation is blocked**; move that logic outside the container. Source: [Code Node guide](https://developers.snap.com/lens-studio/features/graphics/materials/material-editor/code-node/code-node-guide).
- **Overrunning OIT layers** — beyond the selected layer count, extra layers render unsorted (artifacts return); Two-sided materials silently consume extra layers.
- **Exceeding light limits** — only one light casts shadows at a time, and only Directional/Spot lights can; adding more directional/point lights than the 3+5 cap won't all render.
- **Ray Tracing silently no-ops on unsupported devices** or when a caster mesh exceeds **65,000 triangles** — gate it on device capability and keep caster meshes low-poly.
- **Citing a stale doc URL** — the `/4.55.1/...` paths are frozen; use unversioned `/lens-studio/...` for current behavior.

## Custom code-node shaders & material wiring — verified the hard way

From building ~15 full-screen post-effect lenses on the `CodeNodeMaterialPreset` (custom GLSL node) pipeline, LS 5.22.1, Jul 2026.

### Driving a shader from script
- A `.graphShader` custom-code node exposes `input_float` / `input_vec2` / `input_texture_2d` params wired to graph parameter nodes; script sets them as `material.mainPass.<scriptName>`.
- ⚠️ **Adding a new parameter means hand-editing the `.graphShader` YAML** (a `nodes_inputs_parameters_texture2d_object` / `..._float` node plus port wiring). To skip that surgery, **pack two values into one float** and decode in GLSL — e.g. `paramA = mode*10 + strength` → `mode = floor(paramA/10)`, `strength = paramA - mode*10`; or `floor(hue*360) + sat` for a hue+saturation pair. Works for any "index + continuous knob" pair.
- Expose a texture's dimensions to GLSL by setting `ExtraOutputs: true` on its parameter node (then `TextureSize` is available).
- GLSL edits **hot-reload** on preview refresh. Iterate every shader variant inside ONE open project, then clone the project per effect — much faster than a project switch per iteration.

### Binding textures / colors to materials via the API
- ⚠️ The asset-graphql property path is **`passInfos.0.baseTex`**, not `baseTex` (which errors "Property not found"). Value = the texture's GUID with `valueType: REFERENCE`; read that GUID from the PNG's generated `.meta` (`Texture: !<reference> <guid>`).
- ⚠️ **Every image that needs its own texture needs its own material.** `ScreenImageObjectPreset` instances share one `Image.mat`; setting `mainPass.baseTex` (or `baseColor` for runtime tinting) on a shared material changes *every* image using it. Create one `ImageMaterialPreset` per element.
- Runtime tinting uses the same route: `img.mainPass.baseColor = new vec4(r,g,b,1)` on a per-element material.
- A material's `PassesInfo` only exists on disk **after** `project.save()` — save before hand-editing a `.mat`.

### Look-development notes that repeatedly mattered
- **Glow/halo quality:** sampling a mask at a *single* radius gives a hard, banded, "geometric" ring. A **multi-ring gaussian** (e.g. 5 radii × 16 angles, weights `exp(-r²·k)`, normalized) produces the soft cloud people expect — the single biggest visual-quality fix across the suite.
- **Fisheye:** a tan-law radial warp *magnifies* the center and reads as mere edge distortion. The phone-ultra-wide look people actually want is the **inverse** (`atan`) law — center minified, edges stretched. Do the warp in **axis-normalized space** (elliptically symmetric); per-direction rectangle normalization introduces visible **seam bands** along the diagonals. Scale display space with zoom so the lens's dark corners crop away as you punch in, matching real optics.
- **Segmentation:** a Body `SegmentationTexture` bound as a second texture gives `person` ∈ 0..1 for compositing (`mix(effect, cam, person)`); blur it for silhouette glows.
- Choose preview media that actually contains what the effect keys off — a "sparkle" that blooms highlights looks broken on flat lighting and sings on point lights. Not every dull result is a bug.

## Go deeper

- [Materials Overview](https://developers.snap.com/lens-studio/features/graphics/materials/overview) — the authoritative list of base types, properties, and blend modes.
- [Order-Independent Transparency](https://developers.snap.com/lens-studio/features/graphics/advanced/order-independent-transparency) — when and how to enable OIT.
- [VFX Editor Overview](https://developers.snap.com/lens-studio/references/vfx-editor/overview) and [Introduction & Concepts](https://developers.snap.com/lens-studio/features/graphics/particles/vfx-editor/introduction-and-concepts) — node-based particle simulation.
- [Light and Shadow](https://developers.snap.com/lens-studio/features/graphics/light-and-shadow) — light types, Shadow Mapping vs Projective Shadows, and limits.
- [Ray Tracing guide](https://developers.snap.com/lens-studio/features/graphics/raytracing/raytracing-guide) — reflections/GI setup and device support.
- [Performance Optimization Guide](https://developers.snap.com/lens-studio/publishing/optimization/performance-optimization-guide) and [Texture Optimization](https://developers.snap.com/lens-studio/publishing/optimization/texture-optimization) — the budgets every rendering choice must fit inside.
- [ar.snap.com/lens-studio-v5](https://ar.snap.com/lens-studio-v5) — versioned release notes; LS 5.22 brought YAML graph files, Pause When Not Visible, Particle Orient Node.