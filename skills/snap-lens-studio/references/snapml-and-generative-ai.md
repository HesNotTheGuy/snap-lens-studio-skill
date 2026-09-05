# SnapML & Generative-AI Authoring in Lens Studio

Lens Studio ships two distinct machine-learning capability families, and keeping them mentally separate is the single most important thing when working in this area. **SnapML** is *bring-your-own-model* custom ML: you train a model somewhere else (PyTorch, TensorFlow, Roboflow), import it as `.onnx` or `.tflite`, and run it on-device through an **ML Component**. It arrived in **Lens Studio 3.0**. The **GenAI Suite** is a set of Snap-hosted, cloud-backed *authoring* tools inside Lens Studio that turn text/image prompts into ready-to-ship assets or ML effects; it graduated from Beta with **Lens Studio 5.0**, and under the hood its generators are themselves "Powered by proprietary SnapML technology." The mental model to hold: SnapML is the *runtime* for classic hand-built models; GenAI is a *content pipeline* that frequently *emits* SnapML models and assets which then run through that same runtime. Sources: [SnapML Overview](https://developers.snap.com/lens-studio/features/snap-ml/ml-overview), [GenAI Suite blog](https://ar.snap.com/blog/genai-suite-lens-studio-5.0).

> **Platform / version context.** Mainline current is **Lens Studio 5.23.2** (Aug 17, 2026; 5.23.0 on Jul 28, 2026). **Spectacles (2024) development is pinned to Lens Studio 5.15.x**, not mainline — do not assume mainline features exist there. Prefer unversioned `developers.snap.com/lens-studio/...` doc URLs over `/x.y.z/...` paths. Sources: [Release Notes](https://ar.snap.com/lens-studio-v5), [ar.snap.com/download](https://ar.snap.com/download).

---

# PART A — SnapML (custom models)

## What is possible

Import your own trained model and use it on-device for: **classification, object detection, style transfer, custom segmentation, ground segmentation, keyword detection, and multi-object detection**; on Spectacles you additionally get **pose estimation**. Snap ships **7 starter templates** — Classification, Object Detection, Style Transfer, Custom Segmentation, Ground Segmentation, Keyword Detection, and Multi Object Detection — plus a **Model Zoo** of pre-built example models downloadable from the in-app **Asset Library** (Object Detection sets: Household Objects, Animal, Sport, Road Attributes; Segmentation: Toilet/Cat/Dog; a Style Transfer example). An **8th template, Multi Class Classification**, was added later specifically to demonstrate quantized models (see [Quantization](#quantization)). Sources: [SnapML Overview](https://developers.snap.com/lens-studio/features/snap-ml/ml-overview), [Model Zoo](https://developers.snap.com/lens-studio/features/snap-ml/model-zoo).

**Model formats.** Exactly two import formats: **`.onnx` (ONNX)** and **`.tflite` (TensorFlow Lite)**. Train in PyTorch, TensorFlow, Roboflow, or anything ONNX-compatible — there is no proprietary Snap format. Source: [SnapML Overview](https://developers.snap.com/lens-studio/features/snap-ml/ml-overview).

**Size budget.** You get **up to 10 MB of SnapML assets**, and — this is the crucial optimization detail — **Custom ML and Remote Storage assets do NOT count toward the Lens's total size limit** (verbatim: "Custom ML and Remote Storage Assets do not count towards the total size limit"). So a 10 MB model sits *on top of* the base Lens budget (Lens total size target **< 8 MB**; RAM **< 150 MB**). Compression is offered at import time. Sources: [ML Component Overview](https://developers.snap.com/lens-studio/features/snap-ml/ml-component/ml-component-overview), [Performance & Optimization guide](https://developers.snap.com/lens-studio/publishing/optimization/performance-optimization-guide).

## Key components, assets & APIs

**The ML Component** is the runtime object. Attach it to a Scene Object via the Objects panel `+` → **ML Component**, or by dragging the model into the Asset Browser. Inspector settings (all UI-verified):

- **Input** — shape (**Width, Height, Channels**; Width/Height editable, Channels not); an *Input Transformer* (Stretch, Horizontal/Vertical alignment, FlipX, FlipY, Rotation, Fill Color); and **Scale** and **Bias** normalization (per-channel if channels ≤ 4). If channels ≤ 4 you may feed a **Texture** in directly.
- **Output** — read-only shape, a transform source (pick an Input), Scale/Bias, and a **"Create Output Texture"** button (channels ≤ 4).
- **Component-level** — **Render Order**, **Model** (the MLAsset), **Auto build**, **Auto run**.

Source: [ML Component Overview](https://developers.snap.com/lens-studio/features/snap-ml/ml-component/ml-component-overview).

**Scripting API — `Built-In.MLComponent`.** The API is what lets you push inference *off the critical render path* so the Lens keeps drawing frames. Confirmed methods and signatures:

- `build(placeholders: BasePlaceholder[])` — build once all placeholders are set. Async variant **`buildAsync(placeholders)`** returns a Promise.
- `runImmediate(sync: boolean)` — run once, synchronously or asynchronously.
- `runScheduled(recurring: boolean, startTiming: FrameTiming, endTiming: FrameTiming)` — schedule inference between two frame-timing points.
- `getInput(name)` / `getOutput(name)` — get placeholders; **tensor data is exposed as a `Float32Array` and must be mutated in place.**
- Callbacks: **`onLoadingFinished`**, **`onRunningFinished`**, **`onLoadingFailed(error)`**, **`onRunningFailed(error)`**.
- **`FrameTiming`** enum values: **`Update`, `LateUpdate`, `OnRender`, `None`**.

Sources: [MLComponent API](https://developers.snap.com/lens-studio/api/lens-scripting/classes/Built-In.MLComponent.html), [Scripting ML Component](https://developers.snap.com/lens-studio/features/snap-ml/ml-component/scripting-ml-component).

**The Compatibility Table** is how you diagnose on-device cost *before* shipping. On import (or via the MLAsset Inspector) Lens Studio shows a table listing every layer/Op and whether it is supported across the available inference backends — **CPU, iOS GPU, Android GPU, and the iOS inference accelerator / Neural Engine**. A **green** button means all layers are supported; **red** means unsupported layers (hover the warning to get the offending layer name). Unsupported Ops force **CPU fallback** or block acceleration entirely, and that is the number-one cause of slow ML Lenses. *(The four backend columns are drawn from the Inspector UI and export docs, not enumerated in the Overview body text — treat the exact column set as UI-verified.)* Sources: [SnapML Overview](https://developers.snap.com/lens-studio/features/snap-ml/ml-overview), [Export from PyTorch](https://developers.snap.com/lens-studio/features/snap-ml/ml-frameworks/export-from-pytorch).

## How to build it

1. **Pick a starting point** — a template, a Model Zoo model, or your own trained network. Train in PyTorch/TensorFlow/Roboflow and export to `.onnx` or `.tflite`.
2. **Export correctly (PyTorch → ONNX).** Fix input shapes at export — **you cannot change them after import**. Use **`opset_version=11`** if any layer uses `align_corners=True` (default opset 9 lacks that parameter). Use **`align_corners=True`** wherever you use bilinear interpolation — *but* the **iOS inference accelerator does not support `align_corners=True`**, so prefer nearest-neighbor or transposed convolution if you need that accelerator. Inspect the graph with **Netron** to catch unsupported ops by name. Source: [Export from PyTorch](https://developers.snap.com/lens-studio/features/snap-ml/ml-frameworks/export-from-pytorch).
3. **Import** the model. Read the **Compatibility Table** — resolve red/unsupported layers before proceeding. Accept compression if offered.
4. **Attach an ML Component** and wire up the model. **Lens Studio textures live in `[0,255]`;** if your model was trained on whitened `[-1,1]` or `[0,1]` input, set the correct **Scale/Bias** in the ML Component or your colors/results will be wrong.
5. **Drive inference from script.** Use `getInput`/`getOutput`, mutate the `Float32Array` in place, `build()`/`buildAsync()`, then `runImmediate(false)` or `runScheduled(...)` for async, off-frame execution. Hook `onRunningFinished` to read results and `onLoadingFailed`/`onRunningFailed` to handle errors.

<a name="quantization"></a>
**Quantization.** SnapML supports **quantized models** via a **post-training quantization pipeline in Lens Studio** (Python libraries) and quantized-model support **in the TensorFlow Lite importer**; the **Multi Class Classification template** (with quantized training code on GitHub) demonstrates the flow. Stated benefits are qualitative — **"fast inference speed," "small model size," "increased power efficiency"** (from reduced size plus faster fixed-point computation and less memory bandwidth). There is **no specific size-reduction percentage** in the docs (do not claim "~50% smaller"), and the exact Lens Studio minor version that introduced quantization is **not stated** (5.x, open flag). Sources: [SnapML Quantization Support](https://ar.snap.com/snapml-quantization-support), [Multi Class Classification template](https://developers.snap.com/lens-studio/features/snap-ml/snap-ml-templates/multi-class-classification).

## Best practices

- **Limit the Lens to one ML Component** for optimal performance (official recommendation, verbatim).
- Target **30 FPS**; keep it **above 15 FPS** on most devices; high-end hardware can reach ~60 FPS.
- Use tiny/small model variants (e.g. **YOLOv7-tiny**) and reduce input resolution (e.g. **224×224**) to trade quality for speed.
- **Quantize**, and keep the model **< 10 MB**.
- Run inference **async / off-frame** (`runImmediate(false)` or `runScheduled`) whenever every-frame realtime is not required.
- Verify the **Compatibility Table is green** before shipping.

Sources: [Performance Optimization guide](https://developers.snap.com/lens-studio/publishing/optimization/performance-optimization-guide), [SnapML on Spectacles](https://developers.snap.com/spectacles/about-spectacles-features/snapML).

## Common pitfalls

- **Red Compatibility Table → CPU fallback → frame drops.** Unsupported Ops are the main performance killer; resolve them at export/import time.
- **Wrong Scale/Bias → garbled output or wrong colors.** Symptom of a `[0,255]`-vs-`[-1,1]`/`[0,1]` mismatch between Lens Studio textures and how the model was trained.
- **Unsupported Op → import warning.** Use Netron to find it by name before you import.
- **Trying to change input shape after import** — impossible; shapes are fixed at export.
- **Multiple ML Components** — degrades performance; consolidate to one.
- **Publishing lag:** "It might take longer for your SnapML Lenses to be reviewed." SnapML Lenses deploy to **Snapchat, Camera Kit, and Spectacles** (the Overview's "Supported on" badge lists only Snapchat + Camera Kit; Spectacles support is documented on its own page). Sources: [SnapML Overview](https://developers.snap.com/lens-studio/features/snap-ml/ml-overview), [SnapML on Spectacles](https://developers.snap.com/spectacles/about-spectacles-features/snapML).

## SnapML on Spectacles (2024) — key divergences

Spectacles supports **object detection (YOLOv7), classification, semantic segmentation, style transfer, and pose estimation**. Keep the model **under 10 MB** and prefer low-op-count models. **Limit detection to 3–5 objects simultaneously** (up to ~10 with adequate performance). **NMS (non-max suppression) is excluded from the ONNX graph and re-implemented in Lens Studio using JavaScript.** Spectacles 2024 is **pinned to Lens Studio 5.15.x** — do not assume mainline behavior. Source: [SnapML on Spectacles](https://developers.snap.com/spectacles/about-spectacles-features/snapML).

---

# PART B — GenAI Suite (generative-AI authoring, Lens Studio 5.x)

## What is possible

The GenAI Suite (**Lens Studio 5.0+**) is a set of Snap-hosted generators that turn text/image prompts into Lens-ready assets or trainable ML effects. Generation runs in **Snap's cloud** (account/login required — this is not offline generation); the *output* is what runs on-device. Root: [GenAI Suite blog](https://ar.snap.com/blog/genai-suite-lens-studio-5.0).

### 1. Lens Studio AI / "Easy Lens" (text-to-Lens, Beta)

The live page is titled **"Lens Studio AI (Beta)"** — Beta status is confirmed. Describe a Lens in plain English and **Creator Mode** assembles one from **AI-Enabled Components** ("generate AR experiences through plain text descriptions"). A **Developer Mode** in-editor AI assistant can analyze the scene, debug, fix errors, add Asset Library content and trending music, and — thanks to Lens Studio 5.0's native TypeScript support — work in **JavaScript or TypeScript**; dedicated Developer Modes exist for **VS Code and Cursor**. Hard limits (verbatim): it is **English-only** ("It will not respond" to non-English), and it is **not optimized for back-facing / rear (World) experiences** ("Do not ask Lens Studio AI to build back facing experiences. Lens Studio AI is currently not optimized for it."). Sources: [Lens Studio AI (Beta) / Easy Lens](https://developers.snap.com/lens-studio/features/lens-studio-ai/creator-mode), [Creator Mode](https://developers.snap.com/lens-studio/features/lens-studio-ai/creator-mode), [Developer Mode](https://developers.snap.com/lens-studio/features/lens-studio-ai/developer-mode).

### 2. AI-Enabled Components (Blocks)

These are **Custom Components with attached AI metadata** (an `AiMetadata` markdown file), also called **Blocks**, which Creator Mode reads to place and configure them. Examples: **Background Image, Makeup, Screen Sticker**. They pull generated content via **Asset Providers**: **Image** ("full screen image texture with a fixed **4:7 aspect ratio**" — the unusual 4:7 ratio is correct, not a typo), **Sticker** (texture with opacity), **Sprite** (like Sticker but without the white outline), and **3D Object**. Each provider has an **Asset Style** field, which *is* the generation prompt. Source: [AI Enabled Components](https://developers.snap.com/lens-studio/features/lens-studio-ai/ai-custom-components).

### 3. Face Generator / "ML Face Effect Generation" — the GenAI→SnapML bridge

Prompt or image → a **trainable custom ML face-effect model** that you import and ship. This is the clearest place GenAI *emits* a SnapML model. Three variants:

| Variant | Prompt cap | Preview time | Notable settings |
|---|---|---|---|
| **Enhanced** | ≤ 500 chars | ≤ 5 min | morphs/characters/animal transforms, preserves identity; Reference Strength, Attributes Preservation (hair/headwear), Seed |
| **Advanced** | ≤ 500 chars | ≤ 20 min | highest quality, strongest prompt control, widest range of reference inputs |
| **Original** | ≤ 200 chars | ≤ 15 min | preset categories; Effect Intensity, User Identity (Off/Low/Mid/Max), Use Skin Texture, Use Skin Tone, per-feature controls (Eyes, Mouth, Nose, Ears, Brows, Face Contour, Hair) |

After approving a preview, click **"Train model"** — training takes **up to ~2 hours** (confirmed for Enhanced/Advanced) — then import. Effects are confined to a defined face area: "Any part of your design that extends beyond this area will be cropped"; frontal faces are required. Source: [Face Generator](https://developers.snap.com/lens-studio/features/genai-suite/face-ml-generation).

### 4. Texture Generator

Text prompt (**≤ 200 chars**) → **a single texture**, imported to the Asset Browser inside a package folder named **"Texture Generation Result."** The docs do **not** state PBR-material sets, normal/roughness maps, or tiling/seamless output — treat it as a single generated texture only. *(The launch blog separately markets "Material Generation," but the current Texture Generator doc page describes a single texture.)* Source: [Texture Generator](https://developers.snap.com/lens-studio/features/genai-suite/texture).

### 5. 3D Asset Generation (text-to-3D / image-to-3D)

Generates **a single 3D object** (mesh + textures). Supports a text prompt (with a "Surprise me" random option), an optional image upload (a prompt is still required; square images recommended; no people in source images), and negative prompts. Selectable quality levels trade polycount/texture size against asset size. There is **no multi-object scene reconstruction**. Source: [3D Asset Generation](https://developers.snap.com/lens-studio/features/genai-suite/3dag-generation).

### 6. Other GenAI Suite generators (all 5.x)

AI Assistant, ML Face Effects, Immersive ML, Head Morph / Head Generator, Face Mask, Face Animator, Character Skin Generator, AI Portraits/Clips, Style Generator, Garment/Facial-Hair generation, Lens Icon Generation, Bitmoji Animation Generation, 3D Capture, and material/texture generation. *(The launch blog explicitly names AI Assistant, ML Face Effects, Immersive ML, 3D Asset, Head Morph, and "Face Mask, Texture, and Material Generation.")* Source: [GenAI Suite blog](https://ar.snap.com/blog/genai-suite-lens-studio-5.0).

## Common pitfalls

- **GenAI Suite requires Lens Studio 5.0+.** Easy Lens is **still Beta**, **English-only**, and **not optimized for back-facing / World** Lenses — building world experiences with it will underperform.
- **Face Generator training is slow (up to ~2 hrs) and area-cropped** — design inside the face mask or your work gets clipped.
- **Generators are cloud/hosted services** (login required) — not offline generation. *(Docs describe cloud generation and login flows but do not publish an explicit quota line.)*
- **Don't conflate the two families.** Model Zoo and the 7/8 templates are hand-built classic SnapML models; the GenAI generators are separate hosted services that *emit* SnapML models/assets.

---

## Quick reference — version flags

- **SnapML introduced:** Lens Studio **3.0** (confirmed verbatim).
- **Quantization** (post-training pipeline + TFLite importer support + Multi-Class Classification template): **5.x, exact minor not stated** (open flag). No "~50% smaller" figure exists in the docs.
- **GenAI Suite** (Easy Lens, Face/Texture/3D generators, AI-Enabled Components, Creator Mode) and **native TypeScript**: Lens Studio **5.0**.
- **Spectacles 2024 SnapML:** pinned to Lens Studio **5.15.x**; NMS re-implemented in JS; models < 10 MB; detect 3–5 (up to ~10) objects.
- **Mainline current:** Lens Studio **5.23.2** (Aug 17, 2026).

## Go deeper

- [SnapML Overview](https://developers.snap.com/lens-studio/features/snap-ml/ml-overview) — capabilities, templates, formats, Compatibility Table, publishing.
- [ML Component Overview](https://developers.snap.com/lens-studio/features/snap-ml/ml-component/ml-component-overview) — Inspector settings, Input/Output, size rules.
- [MLComponent API](https://developers.snap.com/lens-studio/api/lens-scripting/classes/Built-In.MLComponent.html) & [Scripting ML Component](https://developers.snap.com/lens-studio/features/snap-ml/ml-component/scripting-ml-component) — build/run methods, callbacks, FrameTiming.
- [Export from PyTorch](https://developers.snap.com/lens-studio/features/snap-ml/ml-frameworks/export-from-pytorch) — opset, align_corners, Scale/Bias, Netron.
- [Performance & Optimization guide](https://developers.snap.com/lens-studio/publishing/optimization/performance-optimization-guide) — FPS targets, size budget, the "does not count" rule.
- [SnapML on Spectacles](https://developers.snap.com/spectacles/about-spectacles-features/snapML) — Spectacles-specific limits and NMS-in-JS.
- [GenAI Suite blog](https://ar.snap.com/blog/genai-suite-lens-studio-5.0) & [Lens Studio AI (Beta)](https://developers.snap.com/lens-studio/features/lens-studio-ai/creator-mode) — the generative authoring tools.
- [Face Generator](https://developers.snap.com/lens-studio/features/genai-suite/face-ml-generation) — the GenAI→SnapML face-effect bridge.
