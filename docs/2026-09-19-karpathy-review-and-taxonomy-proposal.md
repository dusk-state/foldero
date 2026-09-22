# Foldero v2.2.0 — Post-Merge Review & Taxonomy-Depth Proposal

**Date:** 2026-09-19 · **Scope:** review of `main` post-v2.2.0-merge (karpathy-guidelines lens) + a forward-looking proposal for deeper project archetypes. No code changed by this report.

---

## Part 1 — Codebase review

### Sizes are healthy, no bloat found
All 21 `skills/*/SKILL.md` files range 76–164 lines (1677 total); all 17 `references/*.md` files range 20–168 lines (1360 total). Nothing is an outlier. The new files added this session (Artifex, the 3 reference docs, 5 hooks + shared lib) match the density of the pre-existing files. No simplification needed on size/bloat grounds — the anti-bloat discipline held.

### Real bug: `taxonomy-version:` is read but never written

**File:** `hooks/post-tool-use-brain-append.sh:34`
```bash
TAXONOMY_VERSION=$(grep -m1 '^taxonomy-version:' "$BRAIN" | cut -d: -f2- | xargs || echo "unknown")
```

This greps a leaf `BRAIN.md` for a `taxonomy-version:` line to populate the child-rollup table's "taxonomy version" column in every ancestor. **Nothing in the codebase ever writes that line.** `SUITE-CONVENTIONS.md` §18 and `folder-execute/SKILL.md`'s Phase A both describe the INDEX section only in prose ("taxonomy version, most recent major decision, open flags") — no file anywhere specifies the literal on-disk format of BRAIN.md's INDEX section, so `folder-execute` never actually emits a `taxonomy-version:` field when it seeds `BRAIN.md`.

**Effect:** every child-rollup row's taxonomy-version column will read `unknown` forever, on every real run. The column SUITE-CONVENTIONS §18 promises ("child's current taxonomy version") silently never populates. Same root cause likely affects `OPEN_FLAGS` (`grep -c '^- flag:'` — no code path writes a line matching `- flag:` either; flags are described conceptually in INDEX but no `## INDEX` template exists to pin the format).

**Root cause:** the BRAIN.md format was specified conceptually (what fields exist) but never given a concrete template (what the fields look like as text) — unlike every other artifact in this suite (`INDEX.md`'s three sections are pinned by §10; `Project-Scaffold-Templates.md` pins six file formats byte-for-byte). BRAIN.md is the one exception.

**Fix (not applied — flagging per review scope):** add a `## BRAIN.md INDEX template` to `SUITE-CONVENTIONS.md` §18 or a new short reference doc, pinning the literal format:
```markdown
taxonomy-version: 1
latest-decision: (none yet)
open-flags: 0
```
— then update `folder-execute/SKILL.md`'s Phase A to write it on first seed, and `folder-plan`/whichever skill bumps taxonomy version to update the line. This is a small, surgical fix (one template block + one seeding-instruction update), not a redesign.

### Confirmed correct (re-verified with fresh eyes)
- `hooks/lib-mv-parse.sh`'s `HOOK_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"` pattern is robust — resolves relative to the script's own location via `BASH_SOURCE`, not `${CLAUDE_PLUGIN_ROOT}` string-matching, so it can't drift if the plugin root variable ever changes format.
- `verify-signoff-gate.sh`'s fail-closed logic (traced again): `MV_DETECTED=no` → allow; `MV_DETECTED=yes` + unparseable args → deny; root undiscoverable → deny; queued-with-fresh-marker → allow; queued-with-stale/missing-marker → deny. Correct on all five paths.
- `verify-run-lock.sh` / `verify-move-mechanics.sh` fail-open logic: unparseable input → warn to stderr, exit 0 (no denial). Correct, and distinct in the right direction from the fail-closed pair above — no swap.
- The `plugin.json` `"hooks": "./hooks/hooks.json"` fix (needed the `./` prefix) is the *only* place a plugin-manifest path field references another file — no other latent instance of this pattern exists elsewhere in the repo (checked `plugin.json`, `marketplace.json` for other path-valued fields; none found).

### Minor, not blocking
- `lib-mv-parse.sh`'s quoted-arg extraction scans the whole command string for the first two `"..."` substrings, not specifically the ones adjacent to `mv` — a compound command like `echo "start"; mv "/a" "/b"` would silently extract the wrong pair rather than failing safely. Foldero's own skills never emit compound `mv` commands, so this is low-likelihood; already noted in an earlier round's review, unchanged.
- Every hook does `jq` on stdin under `set -euo pipefail` with no guard against malformed/non-JSON input — a parse failure would abort via `set -e` rather than degrade gracefully. Same status: previously noted, not fixed, low risk given Claude Code controls the stdin shape.
- `SUITE-CONVENTIONS.md` §12 ("Every in-folder change is a reversible `mv`") never explicitly mandates double-quoting both paths — the hooks' own comments assert this as a Foldero convention, but the convention itself isn't written down as a hard requirement in the section they cite. Worth promoting from "hook-comment assumption" to an actual stated rule in §12, since it's now safety-load-bearing (the fail-open/fail-closed hook logic depends on it holding).

### Recommendation
Fix the `taxonomy-version`/`open-flags` template gap before this is relied on for real routing decisions — it's silent (no error, just permanently-wrong data), which is the worst kind of bug to leave in a persistent-memory feature. The other three items are genuine but low-severity and already known; no urgency.

---

## Part 2 — Proposal: deeper, niche-specific project archetypes

### The ask
`references/Project-Archetype-Library.md` currently has 7 flat, generic archetypes (~100 words each). You want it to eventually distinguish, e.g., a professional wedding photographer's folder needs (session/edit/catalog/gear/admin/insurance) from a photojournalist's, from an amateur's — at 3–7 levels of depth, across many professions.

### Why the current design can't just grow into this
Flat + generic was a deliberate choice this session (ponytail mode, explicitly reviewed and approved twice). Adding real depth to every niche inside one file means: 7 categories × ~5 sub-specialties × ~3 professional tiers ≈ 100+ full entries if done naively, each wanting its own multi-level tree — that's a 5,000+ word single file, blowing the token-budget convention (§16) by 40×, and directly violating the "keep archetypes lean" principle this build stood on. **Depth and leanness are in real tension here — reconciling them needs a structural change, not more prose in the same file.**

### Proposed structure: base + delta, tiered by load-time

**Tier 0 (unchanged):** `Project-Archetype-Library.md` stays exactly as-is — 7 broad categories, ~120 words each, the first interview-phase filter. This is what keeps Artifex's default path cheap.

**Tier 1 (new, loaded conditionally):** one file per broad category under `references/archetypes/<category>.md` (e.g. `references/archetypes/photography.md`), loaded by Artifex's Phase 2 *only when that category matched* — never all-at-once, so token cost stays proportional to what's actually relevant to this interview, not the whole library. Each tier-1 file has two parts:
1. A **base tree** for the category (e.g. Photography: `RAW/`, `Edits/`, `Delivery/`, `Client-comms/`, `Gear/`, `Admin/`).
2. A short **variant table** — one row per niche, stating only the *delta* from the base, not a re-derived full tree:

   | Niche | Adds | Removes | Typical depth |
   |---|---|---|---|
   | Wedding (professional) | `Insurance/`, `Contracts/`, `Second-shooter-notes/` | — | 5 |
   | Photojournalist | `Wire-submissions/`, `Caption-metadata/`, `Embargo-tracking/` | `Client-comms/` | 4 |
   | Amateur/hobbyist | — | `Admin/`, `Gear/` | 2 |

This keeps each new niche a ~15-word row, not a duplicated 100-word entry — the way to add "product photography" or "sports photography" later is one more table row, not a new prose block.

**Tier 2 (explicitly deferred, don't build yet):** further variation within a niche (e.g. "wedding, solo shooter" vs. "wedding, studio with associates") — only worth adding once Tier 1 has real usage data showing it's needed. Building this speculatively now is exactly the YAGNI case the ladder flags.

### What this needs, concretely
- **No new skill.** Artifex's existing Phase 2 (template match) and Phase 3 (research refinement) already do match-then-refine; they just need to know to open a tier-1 file when a category matches, and apply the base+delta to produce the shown tree.
- **One new interview question, conditionally asked.** Phase 1 gets a single follow-up, asked only when the broad category has tier-1 variants (e.g., after "creative/photography," ask "professional client work, editorial, or personal?") — not a new phase, not six new questions per category.
- **A `references/archetypes/` subdirectory.** New for this repo (references/ is currently flat), but a natural, low-risk extension — no convention violated.
- **Extend the token-budget convention** (already in `SUITE-CONVENTIONS.md`) to cover tier-1 files explicitly: base tree ~120 words (same as today's archetypes), variant table rows ~15-20 words each, cap a tier-1 file at ~10 niche rows before it should itself split.

### Honest recommendation on scope
Don't build this library-wide. Ship **one pilot tier-1 file** — photography, since that's your worked example — with the base+delta schema above, capped at 4-5 niches and depth 4 (not 7). Use it to validate: does the schema actually produce good scaffolds in practice? Do the interview follow-up questions feel natural? Only then decide whether to invest in more categories, based on which ones you (or real users) actually scaffold. Building 20 professions × 7 depth levels upfront, before any usage signal, is the over-engineering this session's own ponytail mode was set up to prevent — the same ladder applies here as it did to every other decision this build made.
