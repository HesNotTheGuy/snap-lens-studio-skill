# Materials, Rendering & VFX

Everything a Lens draws to the screen passes through this layer: **materials** decide how surfaces respond to light, **the render pipeline** (Cameras → Layers → Render Targets) decides what gets drawn and in what order, and **particles/VFX/post-effects** add motion and full-screen style. All of it runs on a mobile GPU inside a hard frame budget, so the recurring theme of this area is *visual richness vs. performance* — nearly every knob here trades one for the other. This file targets **Lens Studio 5.23.x mainline** (current build **5.23.2**, Aug 17 2026); the **Spectacles (2024)** target is pinned to **LS 5.15.4**, so validate any 5.2x rendering/VFX feature against the Spectacles docs before shipping to that device. Prefer the unversioned `developers.snap.com/lens-studio/...` doc paths — the `/4.55.1/...` copies are frozen legacy.

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

### ⚠️ A failed compile is SILENT — check it after every scripted edit
A custom code node that fails to compile shows **no magenta, no console error and no exception**; the material silently falls back and the Lens keeps rendering while doing nothing. Detect it with `material.passInfos[0].getPropertyNames()` — a healthy pass lists its real parameters, a dead one returns exactly `["mainColor"]`. The failed compile is **cached**, so a preview refresh will not clear it: open another project and reopen this one. Causes worth knowing: a **reserved GLSL keyword used as a variable** (`flat`, `sample`, `filter`, `input`, `output`, `layout`, `discard`, and the memory qualifiers `coherent`, `volatile`, `restrict`, `readonly`, `writeonly`), a **backslash anywhere** in a spliced body, calling a helper this harness does not declare, and int/float mixing such as `pow(x, 3)`. See SKILL.md → *Silent failures in custom code-node shaders* for the full table and the mediump-precision traps (out-of-range sentinels, `floor()` shifted by non-representable constants, and hash helpers overflowing to NaN).

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
- **Any threshold or multiply against an ABSOLUTE pixel value makes the effect subject-dependent.** A threshold compared against raw luminance has a usable range that *moves* with the subject, and a `col * tint` multiply has authority proportional to how bright the subject already is — both mean the effect does measurably less for some users than others. Drive selection from a quantity that is uniformly 0..1 for every subject, and blend toward an **additive** term as the input darkens. See SKILL.md → *Controls that are alive in code and dead on screen*.
- **Locating a subject with min/max over a thresholded scan quantises the result** to 1/N and causes a cliff, banding and frame-to-frame popping at once. Use **weighted moments** (mask value as the weight; centroid plus standard deviation) for a continuous answer at the same tap count.
- Choose preview media that actually contains what the effect keys off — a "sparkle" that blooms highlights looks broken on flat lighting and sings on point lights. Not every dull result is a bug.

## Go deeper

- [Materials Overview](https://developers.snap.com/lens-studio/features/graphics/materials/overview) — the authoritative list of base types, properties, and blend modes.
- [Order-Independent Transparency](https://developers.snap.com/lens-studio/features/graphics/advanced/order-independent-transparency) — when and how to enable OIT.
- [VFX Editor Overview](https://developers.snap.com/lens-studio/references/vfx-editor/overview) and [Introduction & Concepts](https://developers.snap.com/lens-studio/features/graphics/particles/vfx-editor/introduction-and-concepts) — node-based particle simulation.
- [Light and Shadow](https://developers.snap.com/lens-studio/features/graphics/light-and-shadow) — light types, Shadow Mapping vs Projective Shadows, and limits.
- [Ray Tracing guide](https://developers.snap.com/lens-studio/features/graphics/raytracing/raytracing-guide) — reflections/GI setup and device support.
- [Performance Optimization Guide](https://developers.snap.com/lens-studio/publishing/optimization/performance-optimization-guide) and [Texture Optimization](https://developers.snap.com/lens-studio/publishing/optimization/texture-optimization) — the budgets every rendering choice must fit inside.
- [ar.snap.com/lens-studio-v5](https://ar.snap.com/lens-studio-v5) — versioned release notes; LS 5.22 brought YAML graph files, Pause When Not Visible, Particle Orient Node.
## Estimating a subject's position from a segmentation mask

A shader with no face-position uniform can derive one by sampling the
segmentation mask on a fixed lattice and taking the mask-weighted centroid. Two
failure modes decide whether it works.

**Cover the subject's range before stratifying within it.** It is tempting to
give every sample its own column, on the reasoning that stratifying x cuts the
variance of the marginal you want while stratifying y only buys coverage. That
is wrong as soon as the subject can appear at different HEIGHTS, because a row
that misses the subject contributes no weight at all - it does not add variance
to a column's estimate, it deletes that column from the estimate. A lattice
whose rows sit outside the usual subject band silently runs on a fraction of its
samples.

Measured for a head-centroid estimator at a fixed 24-sample budget, over subject
centres spanning x 0.34-0.67 and y 0.52-0.92 at three distances:

| lattice | rms error | worst | fell back to centre |
|---|---|---|---|
| 24 columns x 1, rows 0.30/0.49/0.68/0.87 | 0.0427 | 0.179 | 7.5% |
| 8 columns x 3, rows 0.58/0.72/0.86 | 0.0240 | 0.066 | 0.0% |

Same budget, same single texture. Spend samples on covering the range first;
stratify inside it with whatever is left.

Two arithmetic notes. Derive the row index with a power-of-two divisor -
`floor(fi / 8.0)` is exact for the whole loop range, where `floor(k * 0.2)`
silently shifts at mediump. And always give the estimator a floor
(`step(0.06, weightSum)`) plus a sane fallback, because an estimator that snaps
to screen centre when it loses the subject is indistinguishable from a dead
shader.

## Softening a mask boundary, and when not to soften it at all

**Widening a smoothstep window does not widen a mask edge.** A segmentation
edge climbs 0 to 1 across a couple of pixels, so `smoothstep(0.14, 0.86, m)`
lands on the same few pixels `smoothstep(0.30, 0.62, m)` did. To spread a mask
boundary you must blur in SCREEN space - average the mask across a span of
texels along the axis that matters - not stretch the value window.

**A shape mismatch cannot be feathered away.** When the content on the two sides
of a boundary differs in shape rather than tone - a mirrored face half has no
ear where the real head has one - every blend width reads as a drawn contour,
because there is no width at which the two agree. Fill the region instead of
gating it: displace the source coordinate until it lands back inside valid
content, so the neighbouring texture stretches to cover the gap. Make the
displacement proportional to how invalid the sample is, and it is exactly zero
wherever the effect already works. Keep a soft validity check on the displaced
coordinate as a fallback for the extreme case.

## Review every state a control can reach

A lens whose control cycles N states has N renders, and checking the default is
checking 1/N of the lens. A defect that is invisible in the state a lens boots
into will ship. Exercise the control through its full cycle during review, on a
video source - and note that a still preview freezes both time and redraw, so
anything animated or interactive must never be validated on one.

## A curve duplicated in JS and GLSL is NOT the same arithmetic

When the same easing curve exists twice - once in the control script driving a
uniform, once in the shader as a fallback - the two run at different precisions,
and that difference can turn one copy into NaN while the other is fine.

A real case: `q = (p - 0.18) / 0.82; q = 1.0 - pow(1.0 - q, 2.2);`

- **JS, float64**: `1.0 - 0.18` is `0.8200000000000001`, one ulp LARGER than the
  literal `0.82`. At `p = 1.0` this gives `q = 1.0000000000000002`, so `1.0 - q`
  is **negative**, and `Math.pow(negative, 2.2)` is **NaN** by spec.
- **GLSL, float32**: `1.0 - 0.18` and `0.82` are the *same* float. Base is `+0.0`,
  `pow(0.0, 2.2)` is `0`. Correct.

The shader copy is right by rounding luck. Reading the two side by side is
therefore actively reassuring - they look identical and one of them is broken.

Rules that follow:
- Write a denominator as the expression that matches its numerator's endpoint
  (`(1.0 - 0.18)`), never as a decimal literal that is "the same number".
- **Clamp any `pow` base that was produced by subtraction.** A base meant to
  reach exactly 0 will overshoot to -1e-16 about half the time.
- A NaN written into a material uniform is worse than a wrong number: GLSL
  `min`/`max`/`clamp` with a NaN operand is implementation-defined, so it either
  poisons the whole frame or silently collapses the control to zero - and which
  one you get depends on the GPU, so it may never reproduce on the machine you
  test on.

## Blurring: three rules that each cost a wasted pass to learn

- **A ring is not a blur.** N taps on a circle of radius r is N discrete probes
  that swing about as the pixel moves; it aliases rather than low-passing. If a
  value must be smooth, tile the area (a 4x4 box) instead of sampling its edge.
- **You cannot smooth away detail finer than your tap spacing.** Killing 2 px
  texture across a 40 px neighbourhood needs hundreds of taps, not sixteen. When
  a smoothing term is not working, check the SPACING before adding taps - and if
  the arithmetic says the tap count is unreachable, change the term that the
  artefact is *linear* in instead. That is usually a weight, not a radius.
- **A mask feather must be finer than the smallest detail it is meant to
  reveal.** Adding fine octaves to an edge does nothing if the mask's smoothstep
  softens across more pixels than the octaves span - the fix gets blurred back
  out and the original artefact appears to survive it.

## Scale a jitter-producing weight with the feature size it perturbs

If an artefact's size in PIXELS goes as `weight / featureCount`, a fixed weight
makes the coarse setting several times worse than the fine one. That matters
disproportionately when the lens BOOTS into the coarse setting, because the
first thing every user sees is the worst state the lens can produce. Scale the
weight so the artefact is constant in pixels across the control's whole range.

## A blend weight cannot carry authority past 1.0

When a control feels weak at the top, the tempting fix is to raise the ceiling
on its blend weight: `mix(col, target, clamp(mask * mix(0.55, 1.45, s), 0, 1))`.
That is worse than doing nothing. The expression is clamped, so wherever `mask`
is 1 the weight saturates at `s = 0.5` and the entire upper half of the travel
becomes **bit-identical** - measured at 0.00/255 across deep, medium and light
skin on a real lens. The "fix" converted a top half that was merely hard to
distinguish into one that was exactly the same.

You cannot blend more than fully. If a control needs more authority than a full
blend, it has to move the **target**, not the weight:

- keep the weight ending at exactly 1.0, reached at `s = 1`, never clamped
- put the recovered range into the target colour - effect depth, tint strength,
  the size of an additive shift

**Check for this by pattern, not by eye:** any `clamp(k * mix(a, b, s), 0, 1)`
where `b > 1` has a dead zone starting at `s = (1 - a) / (b - a)` wherever
`k = 1`. It is visible in the source without rendering anything.

### Scaling an "approximately neutral" constant scales its error

A vector documented as "near luma-neutral (+0.011)" was fine while it was
applied once. Multiplying it to recover lost control travel multiplied the error
with it, reaching +0.028 luma - i.e. the recovered travel went into brightness,
which was the one thing the design forbade. **Any constant that is about to
become scalable must first be solved for its invariant, not eyeballed.**
Solve `0.299R + 0.587G + 0.114B = 0` and the vector stays neutral at every
scale. Doing so here also *increased* the useful travel on dark skin.

## Never label a measurement capture by the state you think you set

Verifying an effect means comparing it against itself disabled, which means
toggling something and capturing twice. The trap is labelling those captures
from your own bookkeeping - "I tapped the toggle, so this one is OFF."

Toggle state tracked by counting injected taps **drifts**. A tap that misses, an
extra one, a preview reset between subjects, and the labels invert. And the
failure is silent: nothing errors, the numbers look plausible, and the
conclusion comes out backwards. In one real case this produced "the effect
lightens skin on 96% of the area, the logic is inverted" for a lens whose logic
was entirely correct - a retune of the working code was minutes away.

**Derive the label from the pixels instead.** Pick a property only the enabled
state can have, and let the data say which capture is which:

- a warm/tan effect -> the enabled frame has the lower G/R ratio on skin
- a desaturating effect -> the enabled frame has lower max-minus-min per pixel
- an overlay or mask -> the enabled frame has pixels the disabled one cannot

A self-labelling measurement cannot drift. It also survives the reviewer asking
"are you sure that was the ON frame?", which a tap count never does.

Two supporting habits:
- **Measure on a STILL source.** With video, subject motion between two captures
  adds a difference floor that has nothing to do with the effect - one such
  comparison read 12.9/255 whole-frame before motion was ruled out. If the
  control is a runtime uniform write, confirm the toggle registers on the still
  first, since some projects do not receive runtime writes in preview.
- **Look at a spatial diff, not just an aggregate.** Render darker-vs-lighter as
  two colours over a dimmed original. "95.9% darker with a thin lighter band at
  the collar" is a claim about the effect; "mean 4.03/255" is not. The map also
  proves the untouched regions really are untouched.

### State what your test could not cover

The Lens Studio preview library's people span roughly luma 0.42-0.56. It has no
genuinely deep-skin subject (~0.15-0.25) and no very fair one. A lens whose
effect depends on subject luminance - anything multiplicative, anything with a
darkness-dependent blend - is therefore **not** validated across real skin tones
by testing on the bundled previews, however many of them you use. Say so in the
lens notes rather than implying full coverage.

## A skin effect will hit teeth and eyes hardest - check them first

`body - garment - hair` is not skin. It still contains **enamel, sclera, lips and
gums**, and a multiplicative skin effect does not treat them gently - it treats
them WORST. A multiply toward a colour moves a pixel in proportion to how bright
it already is, and enamel and sclera are the brightest things on a face. One
measured case: a tan lens shifted teeth -35.8 luma and sclera -26.7 while moving
the cheek it was aimed at by +1.0 - roughly 36x harder on teeth than on skin.

This is the same luminance-proportional-authority property that makes a multiply
too WEAK on dark skin, showing up at the other end of the range. If a lens has
one problem it usually has both, so check both ends.

**Neither brightness nor saturation alone will separate enamel from skin.**
Measured across three subjects: eye-region saturation (0.354-0.497) overlapped
skin (0.324-0.640), and on one subject the forehead was *less* saturated than the
eye region. Brightness alone fails the other way - a lit fair forehead is as
bright as enamel.

The **pair** separates cleanly, because at comparable luma enamel and sclera sit
near 0.28-0.30 saturation where skin sits at 0.43+:

```glsl
float mxC = max(cam.r, max(cam.g, cam.b));
float mnC = min(cam.r, min(cam.g, cam.b));
float satC = (mxC - mnC) / max(mxC, 0.001);
float lumC = dot(cam.rgb, vec3(0.299, 0.587, 0.114));
float enamel = smoothstep(0.58, 0.74, lumC) * (1.0 - smoothstep(0.26, 0.40, satC));
skinMask = skinMask * (1.0 - enamel);
```

Two things this gets right for free: lips and gums are saturated, so they keep the
effect as they should; and specular highlights on skin are bright and neutral, so
they are partly spared - which is arguably correct, since a highlight is light
rather than skin colour, and tinting it is part of what makes a filter look
painted on.

Verify on a SMILING subject with a still source. A closed-mouth idle clip cannot
show the defect at all, and on video two runs sampled at different moments will
put lips where you think teeth are - one such cross-run comparison reported the
fix had made teeth *worse* when it had not.
