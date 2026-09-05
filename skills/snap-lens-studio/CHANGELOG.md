# Changelog — snap-lens-studio skill

**Stamp:** `2026-09-05.1`

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

There is no third “true” copy until GitHub is wired. Until then the two home
directories must be edited **in the same turn**.

## Before you edit

1. Read the stamp in **both** homes (and `git log -1` on GitHub if a remote exists).
2. If stamps differ, diff the two trees and merge into both before adding work.
3. The lesson you are adding has **one home** (usually a file under `references/`).
   Do not paste it into SKILL.md *and* a reference. SKILL.md only gets a router
   line if agents would otherwise miss the file.

## After you edit

1. Bump the stamp (`YYYY-MM-DD.N`) and add a dated entry below.
2. Write the **same files** to the other agent home.
3. If GitHub exists: commit, tag with the stamp, push.
4. Tell the user both copies (and GitHub, if any) now share the new stamp.

## Known drift (do not ignore)

None as of `2026-09-05.1`. Grok MCP/icon lessons and Claude product/API lessons were
unioned. Current live pin is LS **5.23.2** (17 Aug 2026); Spectacles (2024)
remains **5.15.4**.

## Entries

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
