# Low-Fantasy Medieval Character Generator

A single self-contained HTML file — `generator.html` — that generates grim, low-fantasy
medieval characters: full sheet, procedural woodcut portrait, real heraldry with correct
blazon, and an optional AI-written backstory. No build step, no dependencies, no network.
Open it in a browser (double-click works, `file://` is fine) and it runs offline.

Every character is a pure function of a **seed string**. The seed is shown in the UI,
written to the URL hash (`#seed=thornmarch-4417`), and loading that URL reproduces the
character byte-for-byte, portrait and arms included.

## Using it

- **Generate** rolls the seed in the box; **Reroll** picks a fresh seed. `Space` rerolls.
- **Lock & reroll**: every row on the sheet has a padlock (hover a row and press `L`, or
  click it). Locked fields survive rerolls; everything downstream regenerates around them
  and the coherence rules still apply.
- The three dropdowns *constrain* the roll (region / class / sex) rather than forcing a
  value into an otherwise mismatched character.
- **Export**: JSON (full object, re-importable, includes locks), Markdown, a PNG name-card
  (portrait + arms via canvas), and a print stylesheet that fits the sheet on one A4/Letter
  page. `C` copies a share link.
- **Companies**: the **Household** and **Warband** buttons generate a linked group from the
  seed — a strip of portrait cards appears; click a card to view that member's full sheet.
  The share link preserves the mode (`#seed=X&group=household`), so a company reproduces
  exactly like a single character. **Single character** returns to solo mode.

## Households & warbands

A company is a pure function of `(seed, kind)`. Every member runs through the *full*
generation pipeline; shared facts are injected via the same locks-and-overrides mechanism
the UI uses, so all coherence rules still apply to every member:

- **Household**: an adult head (never clergy or outcast), usually a spouse, and any
  children old enough to be characters (14+). All share region, culture, class, and the
  household's current problem. Children's birth order, sibling lists, and father's name are
  constructed from the actual generated household — a son's patronymic byname traces to his
  real father's given name. Widowed heads (≈12%) get no spouse and the children's dead
  parent is marked accordingly.
- **Warband**: a captain (knightly, gentry, or rougher stock) and 3–5 followers with
  distinct roles, all from the same district and all carrying the captain's problem as
  their shared trouble.
- Members get cross-relationships referencing each other's generated names, with tensions
  from dedicated household/warband tension tables.
- The padlocks and the constraint dropdowns apply to **single characters only** — a company
  manages its own internal coherence.

Duplicate given names inside a company can happen and are left alone on purpose: that is
what bynames are for.

## How the pipeline cascades

Generation runs in a strict order (see `GEN.generate`), and each step may narrow the
pools available to later steps:

```
seed → RNG → region → culture → social class → birth → family → sex/age
     → education → occupation → body → mind → the Touch → skills
     → possessions → relationships → secret → problem → name/byname
     → heraldry params → portrait params → validate()
```

Examples of the cascade: the region weights the class table (the free city breeds
burghers, the marches breed knights); class + sex + age + literacy gate the occupation
pool; the occupation biases which scars and ailments the body picks; the byname is
*derived from fields already on the character* (father's name, home settlement, trade,
a matched physical feature, or a matched deed) — never from a free-floating list.

`validate()` (and inline checks during generation) enforce the coherence rules: literate
serfs are clamped or flagged as remarkable, martial scars are stripped from fresh-faced
millers, possessions above the class budget are removed, arms on a peasant are demoted to
a house mark, orphaned minors get a guardian. **Every clamp is logged to `console.debug`**
and shown in debug mode.

The portrait and heraldry are split into *param generation* (runs inside the seeded
pipeline) and *pure rendering* — the blazon text and the drawn shield come from the same
object, so they cannot disagree.

## Debug mode

Open `generator.html?debug=1` (the hash form `#seed=...&debug=1` also works). A panel at
the bottom dumps the full generation trace: every roll with its table and weight, and
every coherence clamp with its rule name.

## How to add a table entry

All content lives in the frozen `DATA` object near the top of the `<script>` — cleanly
separated from the pipeline, which only ever reads it.

1. Find the table (they are grouped and commented by spec section, e.g. `DATA.scars`,
   `DATA.secrets`, `DATA.occupations`).
2. Add an entry in the table's shape. Most tables are weighted:
   `{ w: 3, v: 'a concrete, specific noun phrase' }` — `w` defaults to 1 for bare values.
   Realism lives in the weights; keep the mundane heavy and the dramatic light.
3. Some tables carry tags the pipeline reads — keep them coherent:
   - `DATA.scars`: `loc` (drives portrait overlay placement) and `src` (injury keywords
     matched against occupations' `injuries` for biasing and the fresh-face clamp).
   - `DATA.occupations`: `classes`, `sex`, `lit`, `wealth`, `injuries`, `skills`, `minAge`.
   - `DATA.possessions`: `t` (wealth tier 0–5) and `p` (price).
   - `DATA.touch`: needs both `manifestation` *and* `mundane` — if you can't write a
     plausible mundane explanation, the entry is too high-fantasy for this setting.
   - `DATA.descriptiveEpithets` / `deedEpithets`: `key` is a regex matched against the
     character's own generated text; the epithet is only usable when it matches.

No pipeline changes are needed for ordinary entries; the tables are data, the logic is
elsewhere.

## Setting the API key (optional backstory)

The **Write the tale** button calls the Anthropic API to write a 250–400-word backstory
that hints at the secret obliquely and ends on the unresolved problem.

1. Click **Settings**, paste an Anthropic API key, **Save key**.
2. The key is stored **only in your browser's localStorage** and is sent nowhere except
   `api.anthropic.com`, and only when you press the button. No key → the button is
   disabled with a tooltip saying why.
3. The response streams into the page and is cached against the seed, so rerolling the
   prose is an explicit choice, not an accident of rerendering.

## Acceptance checks

A headless test suite exercises the exact same modules that ship in the file: table
floors, 10/10 seed determinism including SVG output, a 500-character sweep (no exceptions,
no `undefined`, ≥80% peasant-or-below, Touch ≤6%, no unflagged rule-of-tincture
violations, every byname traceable to a field on its own character), blazon/render
agreement for 20 armigerous characters, lock-stability over 20 rerolls, and company checks
(household/warband determinism, shared region/culture/class/problem, child age and
parentage coherence, cross-ties referencing real member names).

## Content summary

12 regions across 6 name cultures (Anglo-, Germanic-, Frankish-, Slavic-, Norse-, and
Frisian-flavoured, 60+ given names each), 64 occupations, and every §3 table at or above
its specified floor.
