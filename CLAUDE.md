# Meal Planning System

## Purpose & mental model
This is a flat-file system — there is no code, no script, no server. Every
operation ("plan my week," "save this recipe," "log feedback," "update the
freezer") is performed by Claude at request time: read the relevant files,
reason, then write/move/edit files directly. Do not propose building a CLI
tool, script, or app for any of this — that is an explicit non-goal.

This repo is shared between two people (synced via GitHub — see "Git sync"
below), so treat the files on disk as possibly stale until you've pulled.

## Directory map
```
meal-planning/
├── CLAUDE.md            # this file
├── freezer.yaml          # current frozen-meal inventory
├── pantry.yaml            # staple ingredients excluded from shopping lists
├── recipes/
│   ├── go-to/             # proven winners -- default pool for weekly planning
│   ├── candidates/        # saved-but-untried, or tried-once-pending-feedback
│   ├── archive/           # retired: disliked, or superseded
│   └── _template.md       # blank schema-correct template -- copy from this when adding a recipe
└── meal-plans/
    └── YYYY-MM-DD.md      # one file per week, named by the date the plan was generated
```

## Invariants (never violate these)
- A recipe's `status` field must always match the directory it lives in. If
  you change status, physically move the file -- don't just edit the field.
- Slugs are stable and unique across all of `recipes/*/`. Since a file can
  move between status directories, find a recipe by slug by checking all
  three directories, not by assuming its current folder.
- `freezer.yaml` quantities never go negative. Remove an item entirely once
  its quantity hits 0.
- A shopping list never includes: pantry staples (`pantry.yaml`, matched
  semantically/fuzzily -- not exact string match), or ingredients for a meal
  that's coming fully-prepared from the freezer.
- Frontmatter `times_made` / `last_made` / `rating` on a recipe are caches
  derived from that recipe's own `## Feedback Log`. If they ever look
  inconsistent with the log, the log wins -- recompute the frontmatter from it.

## Recipe schema
File: `recipes/<status>/<slug>.md`. Frontmatter fields:

| Field | Type | Notes |
|---|---|---|
| title | string | display name |
| slug | string | lowercase-kebab-case, unique across all recipes |
| status | go-to \| candidate \| archive | must match parent directory |
| source | string | a URL, `personal`, or `web-search` |
| date_added | date | |
| last_made | date or null | |
| times_made | int | |
| rating | float or null | rolling average, 1-5 scale, derived from Feedback Log |
| cuisine | list | open vocabulary, see Tag Taxonomy |
| protein | list | open vocabulary |
| meal_type | list | dinner \| lunch \| breakfast \| snack \| side \| dessert |
| effort | quick \| medium \| involved | quick <20 min active, medium 20-45, involved 45+ |
| diet | list | open vocabulary, e.g. vegetarian, gluten-free |
| kid_approved | true \| false \| null | |
| servings | int | |
| freezer_friendly | bool | can this dish itself be frozen (recipe attribute -- distinct from whether it's *currently* in the freezer, which is `freezer.yaml`'s job) |
| tags | list | open catch-all for anything not covered above |

Body sections, in order: `## Ingredients`, `## Instructions`, `## Notes`
(freeform -- substitutions, original pasted caption text if saved from
social media, anything contextual), `## Feedback Log` (append-only, newest
entry at the bottom):
```
- 2026-06-14: rating 5/5 — "kids loved it, cut the cream by half next time"
```

## Freezer schema (`freezer.yaml`)
```yaml
items:
  - recipe_slug: chicken-tikka-masala   # matches a recipe file's slug, or null for untracked leftovers
    label: Chicken Tikka Masala (2 portions)
    kind: meal                          # meal | ingredient -- see below
    quantity: 2
    notes: ""
last_updated: 2026-06-19
```
No freeze-date tracking -- the user doesn't want to maintain that, so don't
add a `frozen_on` field or suggest backfilling one.
`quantity` is a portion/bag count. Using one in a meal plan decrements it by
1; remove the entry entirely if it reaches 0.

`kind` distinguishes two real cases:
- `meal` -- fully prepared, just thaw/reheat. This is a true zero-prep pick
  for "something from the freezer."
- `ingredient` -- a raw protein/component (e.g. plain frozen salmon
  fillets) that still needs a full recipe. It is **not** a zero-prep
  freezer pick -- don't select it to satisfy "one meal from the freezer."
  Instead, when a recipe that uses it gets selected for some other reason,
  drop that ingredient from the shopping list since it's already on hand.

## Pantry schema (`pantry.yaml`)
A flat list under `staples:`. Matching recipe ingredients against this list
is semantic (use judgment -- "kosher salt" matches "salt"), not code-driven
exact matching.

## Meal-plan schema (`meal-plans/YYYY-MM-DD.md`)
Frontmatter: `week_of`, `generated_on`, `constraints_requested` (the user's
request, verbatim), `status` (planned | in-progress | completed).

Body:
```markdown
## Selected Meals

### 1. <Recipe Title> (frozen, if applicable) — <go-to|candidate>
- Why selected: <reasoning, which constraint it satisfies>
- [Recipe](../recipes/<status>/<slug>.md)

## Shopping List
*(deduplicated; pantry staples and frozen-meal ingredients excluded)*
- [ ] item
- [ ] item

## Outcome / Feedback
*(fill in after meals are made, then reconcile into each recipe's own Feedback Log)*
- [ ] <Recipe Title>:
```

## Tag taxonomy (living list -- extend freely, but prefer reusing an existing
value over inventing a near-duplicate; skim a few existing recipe files
before adding a brand new tag)

**Cuisine:** mexican, italian, mediterranean, indian, chinese, japanese,
thai, vietnamese, american, bbq, french, middle-eastern, korean, greek,
british, asian (generic, when a dish doesn't cleanly fit one country)

**Protein:** chicken, beef, pork, fish, shrimp, tofu, beans, eggs, turkey, lamb

**Meal type:** dinner, lunch, breakfast, snack, side, dessert

**Effort:** quick, medium, involved

**Diet:** vegetarian, vegan, gluten-free, dairy-free, low-carb

**Freeform tags:** one-pot, meal-prep-friendly, kid-approved, crowd-pleaser,
spicy, slow-cooker, freezer-friendly, date-night, quick-weeknight

## Algorithms

### "Plan my week" (e.g. "give me 3 meals: one from the freezer, a mexican meal, and something new")
1. `git pull` first (see Git sync) so you're working from the latest state.
2. Parse the request into constraint clauses. Default to 3-4 meals if no
   count is given. Each clause resolves like:
   - "from the freezer" → pick a `kind: meal` item from `freezer.yaml`,
     preferring one whose `recipe_slug` resolves to a real recipe file (so
     you can show full reheating context). Never pick a `kind: ingredient`
     item to satisfy this constraint -- it still needs full cooking.
   - a cuisine/protein/diet/tag constraint (e.g. "mexican," "vegetarian") →
     filter `recipes/go-to/` first. This is the default pool -- the user
     explicitly does not want 3-4 brand new dinners invented every week, so
     reuse proven recipes by default. Fall back to `recipes/candidates/`
     only if go-to has no match for that constraint.
   - "something new" / "try something new" → run the "Find something new"
     algorithm below. Only source a genuinely new recipe when the user asks
     for this explicitly, or when no existing recipe (go-to or candidate)
     satisfies a hard constraint they gave.
   - a specific recipe name ("make the tikka masala again") → direct lookup
     by title/slug across all of `recipes/*/`, skip filtering.
3. Fill any remaining unconstrained slots from `recipes/go-to/`, biasing
   toward variety (avoid repeating the same cuisine/protein twice in one
   week, avoid recipes with a very recent `last_made`) unless told to repeat
   something.
4. For each selected non-frozen meal, gather its `## Ingredients`. Merge
   across meals, combine duplicate lines, exclude pantry staples and
   anything already covered by a frozen meal.
5. Write `meal-plans/YYYY-MM-DD.md` (dated today) with selections + reasoning
   + shopping list + a blank Outcome section.
6. Decrement `freezer.yaml` for any frozen item used (remove the entry if it
   hits 0); update its `last_updated`.
7. Commit and push (see Git sync).

### "Save this recipe" (a pasted URL or pasted text/caption)
- **URL input:** try WebFetch first.
  - If it succeeds (recipe blogs, most non-video sites): extract title,
    ingredients, instructions; create a new file in `recipes/candidates/`
    with `source` set to the URL.
  - If WebFetch fails, or the URL is a known video platform (TikTok,
    Instagram Reels, YouTube Shorts) that won't yield scrapable recipe text:
    never fabricate ingredients or steps from just a title/thumbnail. Ask
    the user to paste the caption/description or describe what they
    remember. If you can at least read page metadata (title/caption), use
    it as a starting point in `## Notes` and leave a `TODO: confirm
    ingredients/instructions` marker in the body until the user fills it in.
- **Pasted text:** parse it directly into Ingredients/Instructions as best
  effort; keep the original pasted text verbatim in `## Notes` for provenance.
- Always: generate a slug from the title, checking for collisions across
  all of `recipes/*/` before writing. Set `status: candidate`, `date_added`
  to today, `times_made: 0`, `last_made: null`, `rating: null`. Infer
  cuisine/protein/tags from the content; ask the user only if genuinely
  ambiguous.
- Commit and push after saving.

### "Log feedback" (e.g. "we made the tikka masala, kids loved it")
1. `git pull` first.
2. Identify the target recipe: by name if the user names one, or by opening
   the most recent `meal-plans/*.md` with unfilled Outcome entries if the
   user just says "log feedback from this week."
3. Append a dated line to that recipe's `## Feedback Log`.
4. Recompute frontmatter: `times_made += 1`, `last_made = today`, `rating` =
   mean of all Feedback Log ratings (simple average unless the user wants
   recency weighted more).
5. Apply the promotion/demotion rule below; move the file between
   directories if status changes.
6. Update the originating meal-plan's Outcome section with the same
   feedback; flip that plan's `status` to `completed` once every meal in it
   has an outcome filled in.
7. Commit and push.

### "Find something new"
- Triggered by an explicit "something new" request, or as the fallback in
  step 2 of "Plan my week" when no existing recipe satisfies a hard
  constraint.
- WebSearch for a recipe matching the requested cuisine/protein/constraint.
  Prefer recipe sites over video platforms (searchability/scrapability).
- WebFetch the most promising 1-2 results for full ingredients/instructions.
- Save the chosen one to `recipes/candidates/` (`source: web-search`)
  **before** presenting it, so it persists even if the user doesn't end up
  making it this week.
- Present it clearly labeled "new -- found via web search."

### Promotion / demotion (guidance, not a rigid gate -- weigh explicit
sentiment in the feedback text over the raw number when they conflict)
- `candidate → go-to`: 2+ feedback entries averaging rating ≥ 4, or one
  entry of 5 with a clearly positive note.
- `go-to → candidate`: rolling rating drops below 3, or the most recent
  entry is strongly negative even if the historical average is still fine
  (recency of a strong negative signal overrides the average).
- `→ archive`: two consecutive entries ≤ 2, or an explicit "never make this
  again."

## Git sync
This repo is shared with another household member via a public GitHub repo
so both copies stay current. Treat git as part of every operation, not a
separate manual step:
- **Before** reading state to plan a week, save a recipe, or log feedback:
  `git pull`.
- **After** any write (new plan file, new/edited recipe, freezer update,
  feedback log): `git add -A && git commit -m "<short description>" && git push`.
- If `git pull` reports a conflict, do not force-resolve destructively --
  surface the conflicting file(s) to the user and let them decide.
