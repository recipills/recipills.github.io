# Recipe Library

A static recipe website — no build step, no dependencies. Hosted on GitHub Pages.

## Structure

```
recipe-site/
├── index.html              ← the entire app (HTML + CSS + JS)
├── recipes/
│   ├── index.json          ← manifest: list of all recipe IDs
│   ├── pasta-carbonara.json
│   ├── chocolate-lava-cake.json
│   └── ...
└── README.md
```

## Setup on GitHub Pages

1. Create a new GitHub repo (e.g. `recipe-library`)
2. Push this folder to the `main` branch
3. Go to **Settings → Pages → Source: Deploy from branch → main / root**
4. Your site will be live at `https://yourusername.github.io/recipe-library/`

> If your repo is in a subdirectory, edit the `BASE` variable at the top of
> the `<script>` block in `index.html`:
> ```js
> const BASE = '/recipe-library'; // match your repo name
> ```

## Adding recipes

### Step 1 — Extract from a book (paste this prompt into Claude)

> **Tip:** Upload the book file (PDF or EPUB) to Claude and paste the prompt
> below. For books with more than ~15 recipes, ask Claude to extract
> programmatically (read the text layer + read any nutrition tables as
> images) rather than transcribing page by page — it is faster, cheaper,
> and far more accurate. Claude can then hand back a zip of JSON files
> instead of pasting them into chat.

```
I'm giving you a recipe book (PDF or EPUB). Extract every recipe and output
them as individual JSON files matching the schema below.

== BEFORE YOU START: scan the whole book ==

Recipe books vary a lot in layout. Do a quick structural pass first and tell
me what you find, THEN extract. Specifically check:

1. WHERE THE MACROS LIVE. This is the most common source of errors.
   The page for each recipe may show only PARTIAL nutrition (e.g. just
   calories and carbs). The full per-serving breakdown (protein, fat,
   sat fat, sugar, fibre, iron) is often in a SEPARATE summary table,
   frequently near the back of the book, sometimes as an image/scan
   rather than selectable text. Find that table and cross-reference each
   recipe to its row BY NAME. If a book has no macros at all, estimate
   them and flag clearly that they are estimates.

2. HOW RECIPES ARE TITLED. Titles may be ALL CAPS, mixed case, split
   across two lines, or contain curly apostrophes / accents. Do not
   assume a single format. Watch out for SUB-HEADINGS that look like
   titles but aren't — e.g. "FOR THE SAUCE", "OVEN METHOD", "TO SERVE",
   "MAKE IT VEGGIE". These belong inside a recipe, not as new recipes.

3. HOW THE BOOK IS ORGANISED vs the schema categories. The book's own
   chapter names (e.g. "Fakeaways", "Batch Cook", "Sides") won't match
   the 9 schema categories. Map each recipe to the closest schema
   category using judgement, not just the chapter name.

== RULES FOR EACH FIELD ==

- `category` must be one of: Breakfast, Lunch, Dinner, Snack, Dessert,
  Drink, Side, Sauce, Other. Infer it; don't expect the book to state it.
- `prepTime` / `cookTime`: integers in minutes, or null. If the book gives
  a range ("15–20 mins") pick the lower or midpoint and be consistent.
  Ignore qualifiers like "plus marinating time" / "plus chilling" for the
  number, but you may note them in `notes`. "No cook" → cookTime 0.
- `servings`: integer or null. Some recipes say "MAKES 24" (e.g. bites,
  cookies) instead of "SERVES N" — convert to a sensible serving count
  and explain the conversion in `notes`.
- `id` / filename slug: lowercase, hyphens only, no apostrophes/accents/
  special characters. Keep it stable and unique.
- INGREDIENTS: capture the MAIN list. Many recipes have secondary groups
  ("For the sauce", "For the topping") — fold those into the same
  `ingredients` array (you may prefix the text, e.g. "Sauce: 2 tbsp …").
  EXCLUDE "to accompany" / "to serve" suggestions that come with their
  own separate calorie counts — those are optional sides, not part of
  the dish's stated macros.
- STEPS: method paragraphs only. Strip out method sub-headers
  ("Oven method", "Air fryer method", "For the chips") — either drop them
  or merge them into the step text. Do not let ingredient lines leak into
  step 1. Keep steps concise; no padding.
- `description`: 1–2 sentences, pulled from the recipe's intro blurb.

== MACROS ==

- Recipe-level `macros` = totals PER SERVING (book table value, or book
  total ÷ servings). Include kcal, protein, carbs, fat, satFat, fibre,
  sugar, and iron (mg). If the table omits a field, estimate and note it.
- Per-INGREDIENT macros are almost never printed in books — estimate them
  from standard nutritional data for the actual quantity listed (not per
  100g). Use 0 for negligible amounts (garnish, seasoning, spray), null
  if truly unknown. These are ballpark estimates.

For EACH recipe, output a separate JSON block like this:

**Filename: `recipes/recipe-slug.json`**
```json
{
  "id": "recipe-slug",
  "title": "Recipe Name",
  "description": "1–2 sentence description of the dish.",
  "cuisine": "Italian",
  "category": "Dinner",
  "tags": ["pasta", "quick", "vegetarian"],
  "prepTime": 10,
  "cookTime": 20,
  "servings": 4,
  "ingredients": [
    {
      "text": "200g chicken breast",
      "kcal": 220,
      "protein": 41,
      "carbs": 0,
      "fat": 5,
      "satFat": 1.4
    },
    {
      "text": "1 tbsp olive oil",
      "kcal": 120,
      "protein": 0,
      "carbs": 0,
      "fat": 14,
      "satFat": 2
    }
  ],
  "steps": [
    "First step as a clear, concise instruction.",
    "Second step."
  ],
  "macros": {
    "kcal": 320,
    "protein": 38,
    "carbs": 12,
    "fat": 14,
    "satFat": 3,
    "fibre": 2,
    "sugar": 4,
    "iron": 2.1
  },
  "notes": "Any tips or variations, or null.",
  "source": "Book Title"
}
```

== VALIDATE BEFORE YOU FINISH ==

Run these checks and report the results:
- Count recipes found. Does it match the book's stated count (often on the
  cover or contents page, e.g. "100 recipes")? If short, list which sections
  you may have missed and re-scan.
- No recipe has 0 ingredients, 0 steps, or kcal of 0 (unless genuinely
  zero-cal, like water-based lollies).
- Every slug in `index.json` has a matching file, and every file is listed
  in the index. No orphans, no dangling entries.
- Spot-check 3–4 recipes against the book pages, including at least one
  whose title spans two lines or has unusual formatting.

After all recipe JSONs, output a final block:

**File: `recipes/index.json`**
```json
["slug-one", "slug-two", "slug-three"]
```

This should be a complete list of ALL recipe IDs (include any I give you
from the existing index).
```

### Step 2 — Save the files

Copy each JSON block Claude outputs into the corresponding file in the
`recipes/` folder (or unzip the bundle Claude provides).

### Step 3 — Update the manifest

Replace `recipes/index.json` with the updated list Claude provides.

### Step 4 — Commit and push

```bash
git add recipes/
git commit -m "Add recipes from [Book Name]"
git push
```

GitHub Pages updates within ~30 seconds.

---

## Extraction notes & gotchas

Lessons from real extractions. Not every book hits every one of these —
treat them as a checklist of things to verify, not assumptions.

- **Macros are frequently split across the book.** Per-recipe pages may
  show only calories (and maybe carbs); the full breakdown lives in a
  summary table elsewhere, often scanned as an image. Always look for a
  second source of nutrition data before assuming a field is missing.
- **Scanned/image tables need visual reading.** If the nutrition table
  has no selectable text, it must be read as an image. Text extraction
  alone will silently drop it.
- **Titles are not reliably uppercase or single-line.** Don't pattern-match
  on "ALL CAPS only". Verify the recipe count after extraction — a parser
  that assumes one title format tends to both *miss* real recipes and
  *invent* fake ones from sub-headings.
- **Sub-headings masquerade as recipes.** "For the sauce", "Oven method",
  "To serve", "Make it veggie" are internal to a recipe. If you see a
  "recipe" with a description but no ingredients, it's probably one of these.
- **Categories must be inferred.** Map the book's chapters to the schema's
  9 categories; the names rarely line up.
- **Servings ≠ always "serves N".** "Makes 24" (bites, biscuits, batches)
  needs converting to a serving count, with the conversion noted.
- **Times come with ranges and qualifiers.** Normalise "15–20 mins",
  "about 1 hr 30 mins", "plus marinating time", "no cook" to clean integers.
- **Secondary ingredient groups and "to accompany" lists differ.** Fold
  component groups ("for the topping") into the ingredients; exclude
  optional accompaniments that carry their own separate calorie counts.
- **Per-ingredient macros are estimates.** Books almost never print them.
  Mark them as ballpark and don't present them as authoritative.
- **Always finish with the validation pass.** Count, completeness, and
  index/file consistency checks catch the majority of extraction errors.

---

## Recipe JSON schema

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | URL-safe slug, matches filename |
| `title` | string | Recipe name |
| `description` | string | 1–2 sentence summary |
| `cuisine` | string | e.g. Italian, Japanese, Mexican |
| `category` | string | One of the 9 categories above |
| `tags` | string[] | Free-form keywords |
| `prepTime` | number\|null | Minutes |
| `cookTime` | number\|null | Minutes |
| `servings` | number\|null | Serving count |
| `ingredients` | object[] | See below |
| `ingredients[].text` | string | "amount + ingredient" |
| `ingredients[].kcal` | number | Calories for that quantity |
| `ingredients[].protein` | number | Grams |
| `ingredients[].carbs` | number | Grams |
| `ingredients[].fat` | number | Grams |
| `ingredients[].satFat` | number | Grams |
| `macros.kcal` | number | Per serving |
| `macros.protein` | number | Grams per serving |
| `macros.carbs` | number | Grams per serving |
| `macros.fat` | number | Grams per serving |
| `macros.satFat` | number | Grams per serving |
| `macros.fibre` | number | Grams per serving |
| `macros.sugar` | number | Grams per serving |
| `macros.iron` | number | mg per serving |
| `steps` | string[] | Ordered method steps |
| `notes` | string\|null | Tips, variations, serving/“makes” conversions |
| `source` | string\|null | Book or origin |

## Features

- Search by name, ingredient, cuisine, tag
- Filter by category (auto-detected)
- Sort by name, quickest, lowest calorie, category
- Per-ingredient macros: kcal · P · C · F · sat fat
- Recipe macros panel: kcal, protein, carbs, fat, sat fat, fibre, sugar, iron
- Serving multiplier (½× 1× 2× 3×) — all macros and ingredient quantities scale accordingly
- Macros labelled as ballpark estimates
