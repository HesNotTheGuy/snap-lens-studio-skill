# Performance, Optimization, Publishing & Policy

Snapchat Lenses run inside a phone camera in real time, so they live under strict, non-negotiable resource budgets. A Lens that exceeds file-size, memory, or frame-rate limits will either be rejected at submission, silently degraded, or crash on lower-end devices. This file covers the exact numeric budgets you must design to, the profiling tools that measure them, the optimization techniques that get you under budget, the publishing/review pipeline, and the content policies that decide whether your Lens goes Live. Every budget here is a *design constraint you plan for from the start*, not something you retrofit at the end — the cheapest optimization is the asset you never imported. **Spectacles Lenses use a completely different performance model** (thermal/power-centric, with a looser [≤ 25 MB published-Lens cap](https://developers.snap.com/spectacles/get-started/start-building/publishing-lens) instead of 8 MB) and are called out separately; never copy mobile budgets onto Spectacles.

Version context: current Lens Studio line is **5.23.1** (Aug 5, 2026). Spectacles (2024) authoring stays pinned to the **5.15.x** series (5.15.4) — verified 2026-08-10, and note the Spectacles *docs* say "Download latest Lens Studio" without naming a version, which does not lift the pin.

## What is possible

- Ship a mobile Snapchat Lens that loads fast, holds frame rate, and stays within RAM on old phones.
- Qualify a Lens as a **Sponsored / AR ad** by meeting the tighter size and activation-time bars.
- Profile a live Lens on a real paired device and inspect per-frame CPU work in **Perfetto**.
- Offload heavy assets to **Remote Assets** / **Lens Cloud** and **SnapML** so they don't count against the 8 MB submission cap.
- Publish through the Lens Publishing Portal, pass the 3-stage review pipeline, and distribute via Snapcode and Lens Link.

## Key components, assets & APIs

- **Budgets to design to (mobile):** Lens file size, RAM, FPS, triangle count, texture dimensions, LAT.
- **Profiling tools:** in-app **bug icon** panel, **Lens Profiler / Mobile Monitor** ("Send to All Snapchat with Lens Profiler" → **Perfetto** traces). The **Lens Performance Toolkit is deprecated**.
- **Cost offloaders:** **SnapML ML Component**, **Remote Assets**, **Lens Cloud**, **Remote Storage Assets**.
- **Optimization levers:** **Performance Texture Compression**, **Draco** mesh compression, **instancing**, **texture atlasing**, **Tween Manager**, **VFX Asset** auto-batching (LS 5.0+), **Scene Manager** async prefabs.
- **Publishing surfaces:** **My Lenses / Lens Publishing Portal**, **Organization**, **Lens Folder** (with roles), **Snapcode**, **Lens Link**.

## How to build it

### Mobile performance budgets (Snapchat phone Lenses)

**File size — hard cap ≤ 8 MB** for the exported/submitted (zipped) Lens ([Performance and Optimization guide](https://developers.snap.com/lens-studio/publishing/optimization/performance-optimization-guide), [Submission Guidelines](https://developers.snap.com/lens-studio/publishing/submitting/submission-guidelines), [Configuring Project Info](https://developers.snap.com/lens-studio/publishing/configuring/configuring-project-info)). Optimizing further below 8 MB improves reach and completion. **Custom ML (SnapML) models and Remote Storage Assets do NOT count toward the 8 MB cap** ([perf guide](https://developers.snap.com/lens-studio/publishing/optimization/performance-optimization-guide)); SnapML has its own **≤ 10 MB** limit ([submission guidelines](https://developers.snap.com/lens-studio/publishing/submitting/submission-guidelines)). **Remote Assets** are **up to 10 MB per asset**, with **up to 500 MB per Organization** of Remote Asset Storage ([Remote Assets Overview](https://developers.snap.com/lens-studio/features/lens-cloud/remote-assets-overview)) — there is no per-Lens "~25 MB" figure; they simply live outside the 8 MB submission bundle. *Why this matters:* moving big textures/audio/models to Remote Assets or SnapML is the single most reliable way to fit a rich Lens under 8 MB.

**Memory (RAM) — ≤ 150 MB** is the authoritative submission/optimization budget ([perf guide](https://developers.snap.com/lens-studio/publishing/optimization/performance-optimization-guide), [submission guidelines](https://developers.snap.com/lens-studio/publishing/submitting/submission-guidelines)). Above **275 MB "is likely to crash lower end devices"** ([Lens QA Troubleshooting](https://developers.snap.com/lens-studio/publishing/optimization/lens-qa-troubleshooting)). Note a genuine live-doc discrepancy: the [Texture Optimization](https://developers.snap.com/lens-studio/publishing/optimization/texture-optimization) page says textures must "consume less than 120 MB of RAM." That 120 MB is real page-specific guidance; **treat 150 MB as the authoritative submission ceiling** and 275 MB as the crash line.

**Frame rate — target 30 FPS**, keep **> 15 FPS on most devices** ([perf guide](https://developers.snap.com/lens-studio/publishing/optimization/performance-optimization-guide)). Submission device thresholds: **≥ 27 FPS on iPhone 6 and higher**, **≥ 15 FPS on Android Galaxy S6 and higher** ([submission guidelines](https://developers.snap.com/lens-studio/publishing/submitting/submission-guidelines)). **FPT (Frame Processing Time)** = CPU time to process one frame, reported alongside FPS in the profiler.

**Geometry** ([perf guide](https://developers.snap.com/lens-studio/publishing/optimization/performance-optimization-guide)): **≤ 100,000 triangles** for standard 3D models; **≤ 60,000 triangles** for meshes with joints/skinning; VFX average budget **~10,000 triangles** ([VFX Graph Optimization](https://developers.snap.com/lens-studio/features/graphics/particles/vfx-editor/vfx-graph-optimization)); rigs **< 100 joints**; animations **< 10 seconds** each.

**Draw calls:** no single hard number — reduce via **instancing**, **texture atlasing**, and material sharing. In LS 5.0+, multiple instances of the same VFX Asset auto-batch into one draw call.

**Textures** ([Texture Optimization](https://developers.snap.com/lens-studio/publishing/optimization/texture-optimization), [perf guide](https://developers.snap.com/lens-studio/publishing/optimization/performance-optimization-guide)): **max 2048 × 2048** per texture (hard); screen-space/full-screen **max 720 × 1280**. Face-effect caps (all on the perf guide): **eye color 64×64**, **lips 250 px** longest side, **lashes/eyeshadow 300 px**, **blush/face paint 450 px**. **Performance Texture Compression** cuts RAM **6× for RGB** and **4× for RGBA**. Downsize 2048 → 1024/512 where you can; sizes within one material need not match; be careful auto-compressing normal/environment maps; use PNG only when you need alpha, otherwise JPG.

**Other caps:** Lens **name ≤ 18 characters** (letters/numbers/spaces only — [project-info](https://developers.snap.com/lens-studio/publishing/configuring/configuring-project-info)); **≤ 10 liquify effects** per render order + face index (else crash); recommended **one ML Component** per Lens. *(A 320×320 Lens icon requirement was previously asserted but is not stated on the project-info or submission pages — treat as unverified.)*

### Sponsored / AR-ad Lenses (extra bars on top of the above)

- File size **< 4 MB** ([Configuring Project Info](https://developers.snap.com/lens-studio/publishing/configuring/configuring-project-info): "please make sure your lens is less than 4MB." The perf guide states only the general 8 MB cap — cite project-info for the 4 MB figure).
- **Lens Activation Time (LAT) < 650 ms** on Snap's benchmark device ([perf guide](https://developers.snap.com/lens-studio/publishing/optimization/performance-optimization-guide): "your Lens must have a Lens Activation Time under 650 ms, as measured on our benchmark device."). **LAT = time from Lens downloaded to first pixel rendered.** *Why:* ads must feel instant, so activation latency is gated even more tightly than file size.

### Profiling on a real device

Real-device testing is **mandatory** (iOS + Android, including older phones).

1. Pair to Snapchat, then tap the **bug icon (top-left)** for a live panel including RAM ([QA Troubleshooting](https://developers.snap.com/lens-studio/publishing/optimization/lens-qa-troubleshooting)).
2. For deep traces use the **Lens Profiler / Mobile Monitor** ([Lens Profiler and Trace Analysis](https://developers.snap.com/lens-studio/publishing/optimization/mobile-monitor)): press **Additional Settings** next to *Preview Lens* → **"Send to All Snapchat with Lens Profiler."** A **red rotating semicircle** around the Mobile Profiling icon means it's recording; traces open in **Perfetto** (open-source) for per-slice inspection.
3. Run QA stress tests: rapid tapping around the scene, holding the record button, repeated Carousel select/deselect, repeated front/back camera swap.
4. Escalate blockers to **lensstudio-support@snapchat.com**.

The **Lens Performance Toolkit is deprecated** — the legacy [/4.55.1/ toolkit page](https://developers.snap.com/lens-studio/4.55.1/sponsored/optimization/lens-performance-toolkit) states verbatim: "This capability has been replaced with Mobile Monitor and no longer works." It formerly rated Size / LAT / Memory / FPT as Good/Fair/Bad; use Lens Profiler / Mobile Monitor instead.

### Optimization techniques (perf guide)

- **Textures:** performance compression (6×/4×), mipmaps to raise FPS (disabling mipmaps on huge textures lowers LAT), reuse identical textures, avoid > 50% wasted texture space, right-size each map.
- **Meshes:** Draco compression, smallest vertex-attribute formats, instancing, atlasing.
- **Materials/shaders:** prefer **Unlit** over Uber/PBR, fewer nodes, static over dynamic params, OIT only when needed.
- **Rendering:** share/limit Render Targets, disable MSAA when not needed, Clear Depth on the first camera/RTT with 3D content.
- **Scripting:** event-driven over per-frame Update, cache values, delete unused objects (they load even when hidden), use async prefabs + Scene Manager, offload to Lens Cloud/Remote Assets.
- **Animation:** prefer **Tween Manager**; avoid Blend Shapes / Vertex Animation.
- **Audio:** MP3 on mobile (WAV on Spectacles), mono, lower bitrate, loop/fade long tracks.

### Publishing & review pipeline

Flow: **Publish → My Lenses / Lens Publishing Portal → Organization → Lens Folder → new/update → Lens details → Upload to Folder or Upload and Publish → Submit.** Review runs in **3 stages: Validation → Automated Testing → Content Review** ([Submitting Your Lens](https://developers.snap.com/lens-studio/publishing/submitting/submitting-your-lens)). Lens statuses: **In Review / Live / Offline / Invalid / Rejected (can resubmit).** Lens Folders carry roles for team access. Distribute via **Snapcode** and **Lens Link**.

### Content policy (rejection categories)

The [Submission Guidelines](https://developers.snap.com/lens-studio/publishing/submitting/submission-guidelines) reject: sexual content, harassment, violence, weapons, misinformation, illegal/regulated goods, hate, and creative-quality failures — including prohibited QR codes, IP/trademark violations, and ChatGPT-integration limits.

## Best practices

- Set your budgets before importing assets; measure LAT/RAM/FPS on a real device continuously, not once at the end.
- Push big models to **SnapML** and big media to **Remote Assets** so the submission bundle stays under 8 MB (and under 4 MB for ads).
- Prefer **Unlit** materials, **Tween Manager** animation, and event-driven scripts by default; reach for PBR/Update loops only when justified.
- Right-size every texture to the face-effect cap; enable Performance Texture Compression early to reclaim 4–6× RAM.
- Keep to **one ML Component** per Lens and test on the oldest devices in your matrix.

## Common pitfalls

- **Assuming SnapML/Remote Assets count toward 8 MB** — they don't; the bundle cap only covers local assets ([perf guide](https://developers.snap.com/lens-studio/publishing/optimization/performance-optimization-guide)).
- **Believing "~25 MB per Lens" of remote data** — wrong; it's **10 MB/asset, 500 MB/Organization** ([Remote Assets Overview](https://developers.snap.com/lens-studio/features/lens-cloud/remote-assets-overview)).
- **Trusting 120 MB as the RAM limit** — that's a Texture-Optimization-page discrepancy; **150 MB** is authoritative, **275 MB** is the crash ceiling.
- **Shipping a Lens as an ad without checking LAT** — ads also require **< 4 MB** and **LAT < 650 ms**; passing the 8 MB cap alone is not enough.
- **> 10 liquify effects** per render order + face index → crash.
- **Hidden objects still load** — unused SceneObjects consume memory even when disabled; delete them.
- **Reaching for the Lens Performance Toolkit** — deprecated and non-functional; use Lens Profiler / Mobile Monitor.
- **Applying mobile budgets to Spectacles** — see below.

## Spectacles Lenses — a different performance model

Authored in Lens Studio but **pinned to 5.15.x** (not the 5.2x mainline), Spectacles uses a **thermal/power-centric** model. Snap explicitly warns "some specifics for Spectacles may contradict these general guidelines" — do not copy mobile budgets.

- **Target ~60 FPS**, render time **under ~16 ms/frame** (1000 ÷ 60 = 16.67 ms) ([Lens Performance Overlay](https://developers.snap.com/spectacles/best-practices/profiling/lens-performance-overlay)).
- **Lens Performance Overlay HUD** shows Lens CPU %, Lens GPU %, Battery, Lens FPS, Lens Render Time, where **100% = target performance budget.** Zones: **< 100%** battery-runtime constrained (target); **100–125%** thermally constrained (may throttle/shorten runtime); **> 125%** severe thermal → very likely thermal standby/shutdown.
- [Optimizing Lens Performance](https://developers.snap.com/spectacles/best-practices/performance-optimization/optimizing-lens-performance) uses a **Lens Power (LP) 0–100 scale** (0 = empty Lens, 100 = max recommended). Textures **512×512 or smaller**; use **WAV over MP3**; limit simultaneous sounds. The Sandbox throttles rendering/FPS under thermal load.

## Publishing details learned in practice

- ⚠️ **Snapchat's auto-generated lens preview clip is only ~3 seconds** (field observation — Snap documents a 9:16 / <32 MB preview video but states no duration, so verify on your own carousel entry). Any lens that auto-cycles modes to advertise itself must fit the whole tour inside that window (e.g. 3 modes × 0.9 s). Lenses with 4–6 modes at ~1.2 s each only ever show a partial tour in the carousel — either speed the cycle up or accept a teaser.
- Tags: max 8, letters/numbers, no spaces (observed in the Publish dialog; not documented, so treat as current-behaviour rather than a contract). ⚠️ **Avoid live brand/trademark names** even when they are the community's own slang for the gear (the skate term for the MK1 fisheye is also an active lens brand; console names like "Game Boy" are owned). Generic descriptors capture the same search intent without review/trademark risk.
- The published lens name comes from the **Publish dialog** — the Editor API's `metaInfo.lensName` does not survive a save.
- A **free + premium sibling pair** works well: ship a simple single-identity lens for reach, and a customizable version (extra modes or interactive UI) as the Snap+ / Lens+ entry.
- Top Performer Payouts are measured in **unique users who post a Snap using your Lens** (in Qualified Regions), counted **within the first 90 days after submission** — a one-shot window, not a rolling one. Snap publishes no numeric threshold, and the qualifying bar moves; check the [Lens Creator Rewards](https://developers.snap.com/lens-studio/monetization/lens-creator-rewards) page rather than trusting a number you read somewhere. Design consequence: breadth of reach beats depth of engagement, and a Lens that launches slowly has spent its window.

## Go deeper

- [Performance and Optimization for Lenses](https://developers.snap.com/lens-studio/publishing/optimization/performance-optimization-guide) — the core mobile budgets and optimization playbook.
- [Submission Guidelines](https://developers.snap.com/lens-studio/publishing/submitting/submission-guidelines) — size/RAM/FPS thresholds and rejection categories.
- [Submitting Your Lens](https://developers.snap.com/lens-studio/publishing/submitting/submitting-your-lens) — publish flow, the 3 review stages, and Lens statuses.
- [Texture Optimization](https://developers.snap.com/lens-studio/publishing/optimization/texture-optimization) — compression, sizing, the 120 MB texture-RAM note.
- [Configuring Project Info](https://developers.snap.com/lens-studio/publishing/configuring/configuring-project-info) — the < 4 MB ad rule and the ≤ 18-char name cap.
- [Lens Profiler and Trace Analysis (Mobile Monitor)](https://developers.snap.com/lens-studio/publishing/optimization/mobile-monitor) — current profiling workflow with Perfetto.
- [Lens QA Troubleshooting](https://developers.snap.com/lens-studio/publishing/optimization/lens-qa-troubleshooting) — bug-icon panel and the 275 MB crash ceiling.
- [Remote Assets Overview](https://developers.snap.com/lens-studio/features/lens-cloud/remote-assets-overview) — 10 MB/asset, 500 MB/Organization limits.
- [Spectacles: Lens Performance Overlay](https://developers.snap.com/spectacles/best-practices/profiling/lens-performance-overlay) and [Optimizing Lens Performance](https://developers.snap.com/spectacles/best-practices/performance-optimization/optimizing-lens-performance) — the Spectacles thermal/LP model.
