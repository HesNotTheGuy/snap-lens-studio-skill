# snap-lens-studio — Lens Studio skill

A skill for building **Snapchat Lenses** in Snap's Lens Studio: face/world/body/hand
tracking, materials & VFX, scripting (JavaScript/TypeScript), SnapML, generative-AI
authoring, UI/audio, physics, performance budgets, and publishing.

**Filters** are static 2D overlays made on the web. **Lenses** are the real-time AR
experiences Lens Studio builds. People often say "filter" when they mean a Lens.

## What it covers

The skill is a **router plus references**. `SKILL.md` has methodology and
cross-cutting notes; nine reference files load on demand, so a materials question
does not pull SnapML into context.

It includes:

- **Delivery target first.** Snapchat, Spectacles, and Camera Kit differ in which
  APIs exist, which no-op, and which Lens Studio version you need.
- **API surface.** Component type strings (`Component.RenderMeshVisual`,
  `Physics.BodyComponent`), script lifecycle and event order, how scripts
  reference scene objects, and methods that are deprecated — including some the
  live docs still show.
- **On-device rendering.** Post-effect and material pipelines, cameras/layers/
  render targets, segmentation masks, custom GLSL nodes, and render-order rules
  that affect compositing.
- **Submission budgets.** The 8 MB cap, 150 MB RAM, the 275 MB crash line,
  triangle/joint/animation limits, texture sizes — including that SnapML models
  and Remote Assets count separately.
- **Review and publishing.** The three review stages, five Lens statuses, and
  content/IP rules that cause rejections (trademarks, QR codes, and similar).
- **Silent failures.** Features that no-op on a given target, magenta materials,
  Lenses that preview but fail on device.
- **Doc lookup.** Which host is canonical, which URL patterns are frozen legacy,
  and how to verify a specific claim against live docs.

Some of the content is not in Snap's docs. It is field notes from shipping
Lenses: preview-panel behavior that does not match device, "supported" paths
that no-op, editor actions that discard work.

## Structure

```
skills/snap-lens-studio/
├── SKILL.md          # router: build methodology, doc-search guide, cross-cutting gotchas, glossary
└── references/       # loaded on demand, not all at once
    ├── fundamentals-and-workflow.md
    ├── scripting.md
    ├── face-tracking-and-effects.md
    ├── world-body-and-tracking.md
    ├── materials-rendering-vfx.md
    ├── snapml-and-generative-ai.md
    ├── interactivity-audio-text-ui-integrations.md
    ├── assets-physics-and-collaboration.md
    └── performance-and-publishing.md
```

SKILL.md is a thin router; the reference files hold the exact APIs, budgets, and property names.

## Install

**Claude Code** — copy `skills/snap-lens-studio/` into either:

- `~/.claude/skills/` — available in every project (on Windows: `C:\Users\<you>\.claude\skills\`)
- `<your-project>/.claude/skills/` — that project only

Then start a new session. It appears as an available skill and auto-triggers on Lens Studio work.
No scripts, no dependencies, nothing to configure.

**Claude.ai / Desktop** — zip `skills/snap-lens-studio/`, rename the zip to `.skill`, and upload it
under Settings → Capabilities → Skills.

## Accuracy

Every non-trivial claim is cited inline to the official docs at `developers.snap.com`. All ten files
were re-checked against live documentation in July 2026; corrections included Face Swap's platform
support, the existence of `InteractionComponent.onTap`, and the Spectacles published-Lens size cap.

Facts are current as of **Lens Studio 5.23.2** (August 2026; re-verified against the live docs on 2026-09-07). Lens Studio ships roughly monthly, so treat version pins as perishable and confirm numbers against the live docs.

## Reporting corrections

Lens Studio ships roughly monthly. APIs get deprecated, docs move, and advice that was true in one
release can stop being true in the next. If something in the skill does not match what you see,
[open an issue](https://github.com/HesNotTheGuy/snap-lens-studio-skill/issues) with what it claimed,
what actually happened, and your Lens Studio version.

Especially worth reporting:

- **A cited claim that is now wrong** — a changed budget, a renamed API, a deprecated method still
  shown as current.
- **A dead or redirected doc link.** These are the skill's citations. One recent example: Snap moved
  release notes off `developers.snap.com`, which broke the instruction for checking whether the skill
  was out of date.
- **A gotcha that is not here yet.** Field notes come from behavior that is not in the official docs.

Past corrections, so the shape is clear: the face-landmark section taught the deprecated
`getLandmark()` polling pattern that Snap's *live docs still show* — caught by checking the bundled
`StudioLib.d.ts` instead, and replaced with `Head.onLandmarksUpdate`. "Bitmoji is unavailable on
Camera Kit" was false. `scene.liveOverlayTarget` was attributed to the wrong API.

Sometimes Snap's own pages disagree. The Spectacles (2024) version pin is stated on the download
surface and repeated on every release page through 5.23.2 — while the Spectacles *setup docs* are
headed "Download latest Lens Studio" and name no version. Following those setup docs installs a
build the 2024 hardware will not run. The skill records which surface to trust and why.

## License

MIT — see [LICENSE](LICENSE).
