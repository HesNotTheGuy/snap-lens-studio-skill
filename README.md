# snap-lens-studio — a Claude skill

A Claude Code skill for building **Snapchat Lenses** in Snap's Lens Studio: face/world/body/hand
tracking, materials & VFX, scripting (JavaScript/TypeScript), SnapML, generative-AI authoring,
UI/audio, physics, performance budgets, and publishing.

Note the terminology it enforces: **Filters** are static 2D overlays made on the web; **Lenses**
are the real-time AR experiences Lens Studio actually builds. Most people say "filter" and mean
"Lens."

## What this actually lets Claude do

Without it, Claude guesses at Lens Studio. It half-remembers an API that was renamed in 5.x,
invents a component type string, or confidently cites a tutorial written for Lens Studio 4. The
failure mode isn't "I don't know" — it's fluent, plausible, wrong.

With it, Claude can:

- **Scope a Lens before building it.** Pick the delivery target first (Snapchat / Spectacles /
  Camera Kit), because that decides which APIs exist, which silently do nothing, and which Lens
  Studio version you must be on. Getting this backwards is the most expensive mistake in the tool.
- **Use the real API surface.** Exact component type strings (`Component.RenderMeshVisual`,
  `Physics.BodyComponent`), the script lifecycle and event order, how scripts reference scene
  objects, and which methods are deprecated — including ones the live docs still show.
- **Build effects that work on a phone, not just in preview.** Post-effect and material pipelines,
  the camera/layer/render-target model, segmentation masks, custom GLSL nodes, and the
  render-order rules that decide whether your effect composites or renders black.
- **Stay inside the budgets that gate submission.** The 8 MB cap, 150 MB RAM, the 275 MB crash
  line, triangle/joint/animation limits, texture sizes — with the traps, like SnapML models and
  Remote Assets counting separately.
- **Get through review.** The three review stages, all five Lens statuses, and the content/IP
  rules that cause rejections — trademarks, QR codes, and the rest.
- **Debug the failures that don't announce themselves.** A feature that silently no-ops on your
  target, a magenta material, a Lens that previews fine and dies on device.
- **Find the right doc instead of hallucinating one.** The skill teaches which host is canonical,
  which URL patterns are frozen legacy, and how to verify a specific claim — so Claude checks
  rather than guesses.

Structurally it's a **router plus references**: `SKILL.md` carries methodology and cross-cutting
gotchas, and the nine reference files load **on demand**, so a question about materials doesn't
drag SnapML into context.

Roughly a third of the content isn't in Snap's docs at all — it's field notes from actually
shipping Lenses. Which preview-panel behavior lies to you, which "supported" path silently
no-ops, which editor action quietly discards your work. Those are the parts that save hours,
and they only exist because something broke first.

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
were adversarially re-verified against live documentation in July 2026; corrections applied included
Face Swap's platform support, the existence of `InteractionComponent.onTap`, and the Spectacles
published-Lens size cap.

Facts are current as of **Lens Studio 5.23.2** (August 2026). Lens Studio ships roughly monthly, so
the skill deliberately teaches verifying against the live docs over trusting remembered numbers —
treat the version pins as perishable, not permanent.

## It gets better when something breaks

This skill is maintained, and the maintenance loop is the point. Lens Studio ships roughly monthly;
APIs get deprecated, docs move, and advice that was true in one release quietly stops being true in
the next. A reference that isn't corrected when it goes wrong decays into confident misinformation —
which is worse than no skill at all.

So: **if the skill tells Claude something that doesn't work, that's a bug worth reporting.**
[Open an issue](https://github.com/HesNotTheGuy/snap-lens-studio-skill/issues) with what it claimed,
what actually happened, and your Lens Studio version. Corrections land in the skill and everyone
using it gets them.

Especially worth reporting:

- **A cited claim that's now wrong** — a changed budget, a renamed API, a deprecated method still
  shown as current.
- **A dead or redirected doc link.** These break constantly and they're the skill's whole evidence
  base. One recent example: Snap moved release notes off `developers.snap.com` entirely, which
  silently broke the very instruction the skill gave for checking whether it was out of date.
- **A gotcha you hit that isn't here yet.** Field notes are the highest-value content, and they
  only come from someone losing an afternoon to something.

Past corrections, so the shape is clear: the face-landmark section taught the deprecated
`getLandmark()` polling pattern that Snap's *live docs still show* — caught by checking the bundled
`StudioLib.d.ts` instead, and replaced with `Head.onLandmarksUpdate`. "Bitmoji is unavailable on
Camera Kit" was simply false. `scene.liveOverlayTarget` was attributed to the wrong API entirely.

Sometimes the correction is that **Snap's own pages disagree with each other**. The Spectacles (2024)
version pin is stated on the download surface and repeated on every release page through 5.23.2 —
while the Spectacles *setup docs* are headed "Download latest Lens Studio" and name no version at
all. Follow the docs literally and you install a build the 2024 hardware won't run. The skill now
says which surface to trust and why, because "check the docs" isn't useful advice when the docs are
the thing that's wrong.

## License

MIT — see [LICENSE](LICENSE).
