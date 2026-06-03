# Recipe Library

A static recipe website — no build step, no dependencies. Hosted on GitHub Pages.

Canonical hosted assets:
- This file: `https://recipills.github.io/README.md`
- Shared ingredient macro table: `https://recipills.github.io/ingredient-macros.json`

> **Using this README to drive an extraction:** you can paste the whole
> Extraction Protocol below into a capable model along with a recipe book, or
> just point the model at `https://recipills.github.io/README.md` and tell it
> to follow the protocol. The protocol is written to be self-contained.

## Structure

```
recipe-site/
├── index.html              ← the entire app (HTML + CSS + JS)
├── ingredient-macros.json  ← shared base-ingredient macro table (reused across books)
├── recipes/
│   ├── index.json          ← manifest: list of all recipe IDs
│   ├── pasta-carbonara.json
│   └── ...
└── README.md
```

## Setup on GitHub Pages

1. Create a repo (e.g. `recipills.github.io` or `recipe-library`).
2. Push this folder to the `main` branch.
3. **Settings → Pages → Source: Deploy from branch → main / root.**
4. Site goes live at `https://recipills.github.io/` (or your repo URL).

> If your repo is in a subdirectory, set the `BASE` variable at the top of the
> `<script>` block in `index.html` to match the repo name.

---

# Extraction Protocol

This is the procedure for turning a recipe book (PDF/EPUB) into recipe JSON
files. Earlier extractions failed in two expensive ways: **recipes were given
the wrong ingredient list** (blocks bled across recipe boundaries on shared
pages), and **titles were mis-read** (sub-headings captured as recipes,
multi-line titles truncated). The protocol below is built specifically to
prevent both. Follow the phases in order; do not skip the verification phase.

> **Recommended model usage**
> - **Manifest building (Phase 0) and Verification (Phase 3) + final
>   reconciliation (Phase 5): use the most capable model available (e.g. the
>   current top-tier Claude Opus).** These steps are cheap to run and
>   catastrophic to get wrong.
> - **Bulk transcription (Phases 1–2): a mid-tier model (e.g. current Claude
>   Sonnet) is acceptable _only_ if a top-tier model has already produced the
>   manifest and will perform the verification pass.**
> - **Default for safety: use the top-tier model end-to-end.** A single book is
>   a small cost next to shipping a wrong recipe, and a single model avoids
>   handoff drift. Reserve the split-model approach for high volume, and never
>   let the cheaper model build the manifest or run verification.

## Phase 0 — Build the canonical manifest from the Contents page (do this FIRST)

Before extracting any recipe content, read the book's **Contents / index pages**
and produce a numbered manifest. This is the single source of truth for *how
many* recipes exist and *what each is called*. Everything downstream is
reconciled against it.

For each recipe, capture: **recipe name**, **chapter/section**, and **page
number** if shown. Output the manifest and state the total count.

- If the book advertises a count (cover/intro, e.g. "100 recipes"), the manifest
  total **must** match it. If it doesn't, find the missing/extra entries and
  resolve before continuing — do not proceed on a mismatched count.
- Treat the manifest names as the authoritative titles. Do **not** later rename
  a recipe based on a heading you see in the body.

## Phase 1 — Locate each recipe; never guess boundaries

For each manifest entry, find that recipe in the body **by its name** (and page
number where available). Record its start position.

- A recipe's content is the span **from its own start up to the start of the
  next recipe in manifest order**. Nothing outside that span may be assigned to
  it. This boundary rule is what stops ingredient blocks leaking between recipes.
- Do not detect recipes by formatting heuristics ("all-caps line"). Titles vary:
  mixed case, split across two lines, curly apostrophes/accents. You already know
  the title from the manifest — anchor to it.

## Phase 2 — Extract within bounds

Within each bounded span, pull: title (from the manifest), prep/cook times,
servings, description, the nutrition block, ingredients, and method steps.

- **Shared pages / variant recipes.** Short recipes (sides, variants, "stuffed
  X" trios) often share a spread, with several `PER SERVING` blocks on one page.
  Split the span on each nutrition block and bind each block to the **nearest
  preceding manifest title**. This is the exact situation that previously caused
  swapped ingredient lists — handle it explicitly.
- **Two ingredient groups.** Recipes with "For the sauce" / "For the topping" /
  "For the platter" have multiple ingredient lists. Fold them all into the one
  `ingredients` array (you may prefix, e.g. `"Sauce: 2 tbsp …"`). Do not keep
  only the first group.
- **Method sub-headers** ("Oven method", "Air fryer method", "To serve") are not
  steps and not recipes — merge into step text or drop.
- **Merged lines.** Text extraction sometimes glues two ingredients onto one
  line (e.g. `"225g butternut squash, peeled and chopped 400g 5%-fat minced
  beef"`). Detect an embedded quantity mid-line and split into separate
  ingredients.
- **Macros that live elsewhere.** Per-recipe pages may show only calories (+
  carbs); the full per-serving breakdown is often in a **summary table near the
  back**, sometimes as a scanned image. Find it and cross-reference each recipe
  by name. If macros are genuinely absent, estimate and flag.
- **Servings.** `"Makes 24"` (bites/biscuits) is not `"Serves N"`. If the
  book's macros are stated *per item* ("per bite"), set `servings` to the item
  count and record it in `notes`; otherwise convert to a serving count and note
  the conversion.
- **Times.** Normalise ranges/qualifiers ("15–20 mins", "plus marinating time",
  "no cook") to clean integer minutes.

## Phase 3 — Verification (the guardrail — do not skip)

For **every** recipe, run these checks and report results:

1. **Boundary check.** The recipe's ingredients and steps physically lie between
   its own manifest title and the next recipe's title. Flag anything sourced
   from outside its span.
2. **Uniqueness check.** No two recipes share an identical (or near-identical)
   ingredient list. Identical lists almost always mean a block was copied to the
   wrong recipe.
3. **Plausibility check.** The ingredient set makes sense for the dish name.
   Flag obvious contradictions (e.g. salmon in a dish titled "stuffed
   mushrooms", meat in a recipe marked vegetarian).
4. **Calorie reconciliation.** Compute `sum(ingredient kcal) ÷ servings` and
   compare to the published per-serving kcal. Investigate anything outside
   roughly **0.5×–1.8×**. Most out-of-band cases are a swapped/partial block or
   a quantity-parsing miss — fix the cause, don't just suppress the flag.
   (Genuinely ultra-low-calorie dishes — clear broths, fruit-and-squash lollies
   — can sit below the band legitimately; confirm by eye.)
5. **Completeness check.** No recipe has 0 ingredients, 0 steps, or (unless
   truly zero-calorie) 0 kcal. Times and servings parsed or explicitly null.

Re-extract any recipe that fails a check, going back to its bounded span in
Phase 1/2. Do not hand-edit a value to make a flag disappear.

## Phase 4 — Estimate ingredient macros from the shared table

Fetch the hosted table: `https://recipills.github.io/ingredient-macros.json`.

- For each ingredient line, parse the quantity + unit, map to the **base
  ingredient** in the table, and scale the table's values by the parsed amount.
  Keying on the base ingredient (not the raw line) means "200g chicken breast"
  and "150g chicken breast" reuse one entry and scale correctly.
- Only **estimate from scratch** for base ingredients not already in the table.
  Collect every new base ingredient (with the per-100g / per-unit / per-tsp
  values you assigned) for the maintenance report in Phase 5.
- Per-ingredient macros are ballpark estimates; treat them as such.
- Table conventions: each entry has a `basis` (`100g`, `100ml`, `tsp`, `tbsp`,
  `unit`/`egg`/`slice`, `fixed`, `tin###`, `carton###`), a `per` array
  `[kcal, protein, carbs, fat, satFat]` for that basis, an optional `avg_g`
  (typical weight of one item when a count is given without grams), and an
  optional `g100` fallback `[kcal,protein,carbs,fat,satFat]` per 100g used when a
  normally-counted item is given by weight (e.g. "350g bacon medallions").
- The current table is tuned for reduced-fat / fat-free products common in
  slimming cookbooks. If a new book relies on full-fat staples, the relevant base
  values may need revisiting — note this in the maintenance report rather than
  silently diverging.

## Phase 5 — Final reconciliation + maintenance report

**Reconcile:**
- Recipe count equals the Phase 0 manifest count (and any advertised count).
- One-to-one mapping between manifest entries and output files — no orphans, no
  duplicates, no missing.
- `recipes/index.json` lists exactly the produced IDs.

**Then output a Maintenance Report** (this is required after every book — the
maintainer applies the updates to the hosted files):

1. **`ingredient-macros.json` changes** — list new base ingredients to add, with
   their `basis`/`per`(/`avg_g`/`g100`) values, plus any existing values you
   believe should change and why. If none, say "no changes."
2. **`README.md` changes** — any new format quirk or failure mode this book
   revealed that future extractions should guard against. If none, say
   "no changes."
3. **Open questions / low-confidence items** — recipes or macros you were unsure
   about, so they can be spot-checked.

Deliver the recipe JSON files (and updated `index.json`) alongside the report.

---

## Recipe JSON schema

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | URL-safe slug, matches filename, unique |
| `title` | string | Recipe name (from the Contents manifest) |
| `description` | string | 1–2 sentence summary |
| `cuisine` | string | e.g. Italian, Indian, British |
| `category` | string | One of: Breakfast, Lunch, Dinner, Snack, Dessert, Drink, Side, Sauce, Other |
| `tags` | string[] | Free-form keywords |
| `prepTime` | number\|null | Minutes |
| `cookTime` | number\|null | Minutes |
| `servings` | number\|null | Serving count (or item count for "makes N") |
| `ingredients` | object[] | See below |
| `ingredients[].text` | string | "amount + ingredient" exactly as written |
| `ingredients[].kcal` | number | Estimated calories for that quantity |
| `ingredients[].protein` | number | Grams |
| `ingredients[].carbs` | number | Grams |
| `ingredients[].fat` | number | Grams |
| `ingredients[].satFat` | number | Grams |
| `steps` | string[] | Ordered method steps |
| `macros.kcal` | number | Per serving (from the book) |
| `macros.protein` | number | Grams per serving |
| `macros.carbs` | number | Grams per serving |
| `macros.fat` | number | Grams per serving |
| `macros.satFat` | number | Grams per serving |
| `macros.fibre` | number | Grams per serving |
| `macros.sugar` | number | Grams per serving |
| `macros.iron` | number | mg per serving |
| `notes` | string\|null | Tips, variations, "makes N"/per-item conversions |
| `source` | string\|null | Book title |

### Example

```json
{
  "id": "recipe-slug",
  "title": "Recipe Name",
  "description": "1–2 sentence description.",
  "cuisine": "Italian",
  "category": "Dinner",
  "tags": ["pasta", "quick"],
  "prepTime": 10,
  "cookTime": 20,
  "servings": 4,
  "ingredients": [
    { "text": "200g chicken breast", "kcal": 212, "protein": 48, "carbs": 0, "fat": 3, "satFat": 0.8 }
  ],
  "steps": ["First step.", "Second step."],
  "macros": { "kcal": 320, "protein": 38, "carbs": 12, "fat": 14, "satFat": 3, "fibre": 2, "sugar": 4, "iron": 2.1 },
  "notes": null,
  "source": "Book Title"
}
```

---

## Adding the files to the site

1. Save each recipe JSON into `recipes/`.
2. Replace `recipes/index.json` with the produced ID list.
3. Apply the Maintenance Report: update the hosted `ingredient-macros.json` and,
   if flagged, this `README.md`.
4. Commit and push:
   ```bash
   git add recipes/ ingredient-macros.json README.md
   git commit -m "Add recipes from [Book Name]"
   git push
   ```
   GitHub Pages updates within ~30 seconds.

---

## Extraction gotchas (quick checklist)

Distilled from real extractions. Verify each per book — not every book hits
every one.

- **Anchor to the Contents manifest, not formatting.** Prevents both missed
  recipes and sub-headings masquerading as recipes.
- **Bound each recipe by the next recipe's start.** Prevents ingredient blocks
  bleeding across recipes — the single biggest past failure.
- **Shared pages need per-block binding.** Multiple `PER SERVING` blocks on one
  spread → bind each to its nearest preceding manifest title.
- **Macros often live in a separate (sometimes scanned) table.** Look for a
  second nutrition source before assuming a field is missing.
- **Multiple ingredient groups → one merged list.** Don't keep only the first.
- **Split glued ingredient lines.** Watch for a quantity appearing mid-line.
- **"Makes N" ≠ "Serves N";** per-item macros need noting.
- **Normalise time ranges/qualifiers** to integers.
- **Reuse the shared ingredient table;** only estimate genuinely new bases, and
  report them back.
- **Run Phase 3 verification every time** — boundary, uniqueness, plausibility,
  calorie reconciliation, completeness.

---

## Features

- Search by name, ingredient, cuisine, tag
- Filter by category; sort by name, quickest, lowest calorie, category
- Per-ingredient macros: kcal · P · C · F · sat fat (ballpark estimates)
- Recipe macros panel: kcal, protein, carbs, fat, sat fat, fibre, sugar, iron
- Serving multiplier (½× 1× 2× 3×) scales all macros and ingredient quantities