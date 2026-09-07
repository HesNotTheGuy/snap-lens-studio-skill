# Changelog — snap-lens-studio skill

**Stamp:** `2026-09-07.1`

This file is the sync key for every agent that loads the skill (Grok, Claude, and
anything cloning from GitHub). Read it **before** editing the skill. If two
copies disagree on the stamp, stop and merge — do not stack a new lesson on a
stale tree.

## Copies

| Location | Agent |
|---|---|
| `~/.grok/skills/snap-lens-studio/` | Grok |
| `~/.claude/skills/snap-lens-studio/` | Claude |
| GitHub | `https://github.com/HesNotTheGuy/snap-lens-studio-skill` (public; owner username only — no catalogue/revenue notes) |

GitHub is wired and is the third copy — the sync point for any agent that clones
rather than reads a home directory. All three must carry the same stamp: edit both
home directories **in the same turn**, then commit, tag with the stamp, and push.

## Before you edit

1. Read the stamp in **both** homes and the newest tag on GitHub.
2. If stamps differ, diff the two trees and merge into both before adding work.
3. The lesson you are adding has **one home** (usually a file under `references/`).
   Do not paste it into SKILL.md *and* a reference. SKILL.md only gets a router
   line if agents would otherwise miss the file.

## After you edit

1. Bump the stamp (`YYYY-MM-DD.N`) and add a dated entry below.
2. Write the **same files** to the other agent home.
3. Commit, tag with the stamp (`git tag YYYY-MM-DD.N`), and push `main` **with tags**.
4. Tell the user all three copies now share the new stamp.

## Known drift (do not ignore)

None as of `2026-09-07.1`. All three copies (Grok home, Claude home, GitHub `main`)
carry this stamp. Current live pin is LS **5.23.2** (17 Aug 2026, re-verified
2026-09-07 — no 5.24 exists); Spectacles (2024) remains **5.15.4**.

## Entries

### 2026-09-07.1 — docs re-verification + review fixes

- Snap docs re-checked 2026-09-07: 5.23.2 still current (no 5.24); Spectacles (2024)
  still 5.15.4; Camera Kit matrix unchanged and **still tops out at LS 5.18.x**;
  Bitmoji badge unchanged; submission thresholds unchanged.
- **Home:** `references/performance-and-publishing.md` — Lens Creator Rewards: the two
  programmes, permanent one-way enrolment, and Snap's **ineligible lens types**
  (basic filters, beautification tools, static 2D images, frames, copies); growth
  hacks named as an explicit rejection reason; the 320×320 icon hedge rewritten.
- **Home:** `references/fundamentals-and-workflow.md` — the MCP token **rotates on
  server restart inside LS** and `.mcp.json` is written to the open project's
  directory; three-way triage for `401` / `Tool not found` / no tools. SVG Text
  corrected to **5.23.1** (5.23.2 is bug fixes only). Stale "5.22" template hedge
  dropped.
- **Home:** `references/interactivity-audio-text-ui-integrations.md` — the Camera Kit
  matrix has no rows for 5.19–5.23 (author Camera Kit Lenses on ≤ 5.18.x); all LTS
  windows closed.
- `SKILL.md`: 320×320 removed from the "unconfirmed" list (it is LS's default icon
  size and the recipe is verified at it); router line for the MCP triage; the
  "contra the note above" hot-reload bullet reworded — it agreed with the bullet it
  argued with.
- `references/face-tracking-and-effects.md`: duplicate "expression weights zero"
  pitfall removed.
- Tags: `2026-09-05.1` applied retroactively to the union commit; this stamp tagged
  on its own commit; `main` pushed with tags.
- **Open, not done:** SKILL.md (~440 lines) still carries five full lesson blocks
  that the one-home rule says belong in `references/` — the materials reference even
  points back to SKILL.md for the shader-failure table. Moving them is a separate
  pass.

### 2026-09-05.1 — union merge + 5.23.2 pin

- Union of Grok and Claude homes (not a wholesale overwrite of either).
- Kept Grok editor/MCP lessons: `setEmptyProject`+`openProject` hard-crash,
  320×320 `iconHash` recipe, no `importExternalFile` of in-project paths,
  GLSL-string hot-reload vs structural graph breakage.
- Kept Claude product/API lessons: Camera Kit Bitmoji = limited (not unavailable),
  Ray Tracing iOS yes / Android no, `Head.onLandmarksUpdate`, `TurnOnEvent`
  deprecated, silent-shader detector, dead-control tests, testing traps.
- Version pin: mainline **5.23.2** (17 Aug 2026). Spectacles (2024) **5.15.4**.
- Release-notes surface is [ar.snap.com/download](https://ar.snap.com/download);
  `developers.snap.com/lens-studio/download/release-notes` does not carry the
  current version.
- Hand Mesh (LS 5.23) noted in `references/world-body-and-tracking.md`.
- Hung-object lesson kept in world-body + SKILL.md testing traps.

### 2026-08-27.1 — hung objects are not UV boxes

- **Home:** `references/world-body-and-tracking.md` (best practice + pitfall).
- **Also:** testing trap on Idle Person 1–6.

### 2026-08-27.2 — split audit (no merge)

- File-by-file diff of Grok vs Claude homes. Merge completed in `2026-09-05.1`.
