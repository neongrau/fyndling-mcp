# fyndling-mcp

[![smithery badge](https://smithery.ai/badge/ralf/fyndling)](https://smithery.ai/servers/ralf/fyndling) [![neongrau/fyndling-mcp MCP server](https://glama.ai/mcp/servers/neongrau/fyndling-mcp/badges/score.svg)](https://glama.ai/mcp/servers/neongrau/fyndling-mcp)

Built for **medieval market fans, reenactors, and living-history enthusiasts** — and the AI assistants that help them plan. Fyndling MCP gives AI clients direct access to two niche European datasets:

- **Medieval events** — query 5,800+ markets, concerts, castle experiences, living-history events, and renaissance faires across 26+ countries in Europe and North America, by **location + radius + date range** (updated weekly)
- **Permanent POIs** — 1,900+ meaderies, mead producers, castles, and medieval restaurants, also searchable by geo-radius
- **Historical recipes** — 4,650+ recipes from **forty cookbooks** spanning c. 1300–1581, with modern German adaptations, structured ingredient lists, original manuscript transcripts, and a controlled tag vocabulary for dish-type, diet, and social-class filtering

→ **[fyndling.de](https://fyndling.de)** — the web app behind this data

**Endpoint:** `https://fyndling.de/mcp`  
**Transport:** Streamable HTTP (MCP spec 2025-03-26)  
**Auth:** none  
**Rate limit:** 60 requests / minute  

---

## Quickstart

Add to your MCP client config (e.g. Claude Desktop `claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "fyndling": {
      "url": "https://fyndling.de/mcp"
    }
  }
}
```

---

## Tools

### Events & Locations

All list tools (`find_events_near`, `list_events`, `find_pois_near`, `search_recipes`) return the same envelope: `total` (all matches), `count`, `offset`, `sort`, `has_more`, `next_offset`, plus the array (`events`, `pois`, `recipes`). Page with `offset`. Orders always tie-break on `id`, so pages never skip or repeat. Errors (unknown id, invalid input) are real MCP errors (`isError`), never normal results.

**Event schema** (fixed; same fields for every source): `id`, `name`, `date_from`, `date_to` (null = single day), `ort`, `plz`, `country`, `lat`, `lon`, `geo_precision`, `venue`, `category`, `categories[]`, `flags[]`, `cancelled`, `source_url`, `fyndling_url`, `description` (excerpt in lists), `extra{}` (source-specific raw fields); optional `band`, `organizers[]`, `duplicate_ids[]`. Searches around a point add `distance_km` and `distance_approximate`.

- `category` is the stored category (`market`, `concert`, `burg_event`, `renfaire`); `categories[]` adds flag-based memberships such as `living_history`. Filters (`category`, `types`) test `categories[]`, so response and filter never disagree.
- `flags[]` includes `living_history`, `beerenweine`, `free_admission`, `sold_out` and the derived `party` (dance and party nights such as "Bal Renaissance", classified as `market`).
- `geo_precision` is **derived, not verified**: `venue`, `address`, `plz` or `city`. Coordinates shared by many venues (e.g. one point per city) count as `city`. Imprecise events are shown and marked (`distance_approximate`) but sorted as if up to 5 km (`plz`) or 10 km (`city`) farther away, so they never crowd out precise results.
- Placeholder texts from scraping are returned as `null`.

#### `find_events_near`

Find medieval events near a geographic coordinate, sorted by effective distance, then date, then id.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `lat` | number | ✓ | Latitude, -90 to 90 |
| `lon` | number | ✓ | Longitude, -180 to 180 |
| `radius_km` | number | — | Search radius in km (default 50, max 500) |
| `date_from` | string | — | `YYYY-MM-DD`, a real calendar date. **Default: today (Europe/Berlin)** |
| `date_to` | string | — | `YYYY-MM-DD`, not before `date_from` |
| `types` | array | — | `market`, `concert`, `burg_event`, `renfaire`, `living_history` |
| `limit` / `offset` | integer | — | Page size (default 20, max 100) / skip |
| `include_cancelled` | boolean | — | Default `false`. Cancelled = DB flag or "abgesagt" in name/description |
| `dedupe` | boolean | — | Default `true`: hide display duplicates (same category, normalised name, date and place); the kept entry lists `duplicate_ids`. IDs and scraper keys are untouched |

The date filter tests **overlap**: a multi-day event that started before `date_from` and still runs is found. Recurring markets are one entry showing the next date. The response echoes `effective_date_from` and `effective_date_to`. Invalid input (lat 95, `"morgen"`, reversed range) is a validation error.

**Example — markets within 80 km of Vienna this summer:**
```json
{
  "lat": 48.2082, "lon": 16.3738,
  "radius_km": 80,
  "date_from": "2026-06-01", "date_to": "2026-08-31",
  "types": ["market"]
}
```

---

#### `list_events`

List events filtered by category, country, and/or date range, ordered by start date, then id.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `category` | string | — | `market`, `concert`, `burg_event`, `renfaire`, `living_history` |
| `country` | string | — | ISO 3166-1 alpha-2 code (e.g. `DE`, `AT`, `FR`, `PL`) |
| `date_from` / `date_to` | string | — | `YYYY-MM-DD`; `date_from` defaults to today (Europe/Berlin) |
| `limit` / `offset` | integer | — | Page size (default 20, max 100) / skip |
| `include_cancelled`, `dedupe` | boolean | — | As in `find_events_near` |

---

#### `get_event`

Full details for one event (also accepts POI ids): complete description, `organizers`, `other_dates` (same series nearby), `nearby_events` and `nearby_pois` (up to 5 each within 25 km, upcoming only). An unknown or deleted id is an MCP error.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `id` | string | ✓ | Event ID (16-char hex, e.g. `a1b2c3d4e5f6a7b8`) |

---

#### `find_pois_near`

Find permanent medieval-themed locations, sorted by effective distance, then id.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `lat` / `lon` | number | ✓ | Coordinates (validated like above) |
| `radius_km` | number | — | Default 100, max 1000 |
| `poi_type` | string | — | `meadery` (mead producer; `metkellerei` and `metkellereien` are accepted aliases), `burg` (castle), `ma_gastronomie` (medieval restaurant; the country is a separate field). Output `poi_type` is always canonical and equals the filter value |
| `limit` / `offset` | integer | — | Page size (default 20, max 100) / skip |

**POI schema:** `id`, `name`, `poi_type`, `ort`, `plz`, `country`, `state`, `address`, `lat`, `lon`, `geo_precision`, `website`, `website_ok` (`true`/`false` = last link check, `null` = unknown), `collected_at`, `distance_km`, `distance_approximate`, `fyndling_url`, `extra{}` (e.g. `zustand`/`slug` for castles). No event fields.

---

### Historical Recipes

#### `list_recipe_sources`

List all available cookbooks with metadata (year, language, region, recipe count) and the full course taxonomy (atomic values + aliases).

No parameters.

**Sources** (40 active cookbooks, ordered by date):

| Key | Title | Year | Language | Recipes |
|---|---|---|---|---|
| `harpestreng` | Kogebog (Harpestreng-Handschrift NKS 66) | ~1300 | Old Danish | 25 |
| `viandier` | Le Viandier de Taillevent | ~1300 | Old French | 55 |
| `buch-guter-speise` | Das Buch von guter Speise | ~1350 | Middle High German | 101 |
| `corema_n2` | Nürnberg, Germanisches Nationalmuseum, Hs 3227a | ~1389 | Middle High German (East Central German) | 15 |
| `anonimo_toscano` | Anonimo Toscano (Libro della cocina) | ~1390 | Tuscan Volgare | 40 |
| `form-of-cury` | The Forme of Cury | ~1390 | Middle English | 192 |
| `menagier` | Ménagier de Paris | 1393 | Old French | 378 |
| `bockenheim` | Registrum Coquine (Johannes von Bockenheim) | ~1433 | Medieval Latin | 70 |
| `muenchner_cgm811` | Münchner Handschrift Cgm 811 | ~1440 | Early New High German (Swabian–Bavarian) | 4 |
| `corema_ka2` | Haus- und Arzneibuch (Karlsruhe, Cod. Donaueschingen 793) | ~1445 | Early New High German (Bavarian, lower Inn valley) | 56 |
| `rheinfraenkisches_kochbuch` | Rheinfränkisches Kochbuch | ~1445 | Rhine-Franconian (Middle High German) | 76 |
| `corema_ka1` | Reichenauer Kochbuch (Karlsruhe, Cod. Aug. pap. 125) | 15th c. | Early New High German (Alemannic) | 104 |
| `hausbuch-donaueschingen-792` | Haus- und Arzneibuch im Codex Donaueschingen 792 | ~1450 | Early New High German (Alemannic, Lake Constance region) | 2 |
| `corema_b4` | Berlin, Staatsbibliothek, Ms. germ. qu. 1187 | ~1450 | Early New High German (Bavarian) | 269 |
| `corema_w1` | Vienna, Austrian National Library, Cod. 2897 | ~1450 | Early New High German (Bavarian-Austrian) | 234 |
| `meister_eberhard` | Kochbuch Meister Eberhards | ~1450 | Early New High German (Bavarian) | 23 |
| `tegernsee` | Tegernseer Speisenbuch | 1453–1534 | Early New High German (Bavarian) | 51 |
| `meister_hans` | Kochbuch des Meisters Hans (UB Basel, A.N.V. 12) | ~1460 | Early New High German (Alemannic–Swabian) | 289 |
| `muenchner_clm15632` | Klosterkochbuch Rott am Inn (Clm 15632) | 1458/1464 | Early New High German (Bavarian) | 55 |
| `martino` | Libro de Arte Coquinaria | ~1465 | Early Italian | 268 |
| `koenigsberger_kochbuch` | Königsberger Kochbuch (Deutschordensarchiv) | 15th c. | Early New High German (Bavarian / East-Central) | 34 |
| `muenchner_cgm384` | Münchner Handschrift Cgm 384 | ~1470 | Early New High German (Alemannic) | 83 |
| `corema_k1` | Kölner Küchenmeisterei (Historisches Archiv der Stadt Köln, Best. 7004, 27) | 15th–16th c. (disputed) | Early New High German (Ripuarian/West-Central German) | 24 |
| `edelike_spijse` | Von guten und edlen Speisen (Wel ende edelike spijse) | ~1475 | Middle Dutch | 62 |
| `muenchner_cgm467` | Hausbuch aus Dietramszell (Cgm 467) | ~1477 | Early New High German (Bavarian) | 3 |
| `mondseer_kochbuch` | Mondseer Kochbuch (Graz, UB Ms. 1609) | ~1480 | Early New High German (Bavarian–Austrian) | 268 |
| `muenchner_cgm725` | Münchner Handschrift Cgm 725 | late 15th c. | Early New High German (Bavarian) | 22 |
| `corema_so1` | Solothurner Küchenmeisterei (Zentralbibliothek Solothurn, Cod. S 490) | ~1487 | Early New High German (Alemannic) | 26 |
| `corema_n1` | Alemannisches Büchlein von guter Speise | 1492–1494 | Early New High German (Swabian) | 26 |
| `weinbuch-donaueschingen` | Weinbuch im Codex Donaueschingen | ~1500 | Early New High German (South Rhine-Franconian) | 28 |
| `muenchner_cgm349` | Münchner Handschrift Cgm 349 | 16th c. (addendum) | Early New High German (Bavarian) | 4 |
| `muenchner_cgm5919` | Regensburger Kochbuch (Cgm 5919) | ~1505 | Early New High German (Bavarian) | 104 |
| `severin` | Kuchařství (Böhmisches Kochbuch) | 1535 | Early Czech | 147 |
| `corema_wo10` | Wolfenbüttel, Herzog-August-Bibliothek, Cod. Guelf. 42.3-5. Aug. 4° | 1543 | Early New High German | 116 |
| `philippine_welser` | Das Kochbuch der Philippine Welser | 1543 | Early New High German (East Swabian/Augsburg, Tyrolean-Bavarian additions) | 242 |
| `staindl` | Ein künstlichs vnd nutzlichs Kochbuch (Balthasar Staindl) | 1545 | Early New High German (Swabian/Augsburg) | 294 |
| `sabina_welserin` | Das Kochbuch der Sabina Welserin | 1553 | Early New High German (East Swabian/Augsburg) | 206 |
| `stenglerin` | Kochbuch der Maria Stenglerin | 1554 | Early New High German (Swabian/Augsburg) | 144 |
| `koch_kellermeisterei` | Koch und Kellermeisterei | 1574 | Early New High German | 110 |
| `rumpolt` | Ein new Kochbuch (Marx Rumpolt) | 1581 | Early New High German (Frankfurt/Mainz) | 400 |

The **Recipes** column above is the number of recipes *currently available* from each source. `list_recipe_sources` returns two figures per source: `recipe_count` (imported and searchable now) and `recipe_count_original` (recipes in the manuscript or edition, `null` if unknown). Descriptions may quote the original figure, so the two legitimately differ for sources still being ingested. The corpus is reviewed and published source-by-source, so all counts grow over time.

---

#### `list_recipe_tags`

List the controlled tag vocabulary, grouped by category (`dish_type`, `diet`, `social_class`), with labels, descriptions and current recipe counts. Use these tag IDs as values for the `tags` parameter in `search_recipes` and `compose_menu`.

No parameters.

**Tag groups:**

| Group | Tags |
|---|---|
| `dish_type` | `pasta`, `reis`, `brei`, `beilage`, `huelsenfruechte`, `brot` |
| `diet` | `vegetarisch`, `vegan`, `fastenspeise` |
| `social_class` | `hofkueche`, `buergerlich`, `bauernkueche` |
| `beverage` | `met`, `methiglyn`, `bochet`, `melomel`, `hippocras` |
| `occasion` | `festtag`, `schauspeise` |

The response also lists `uncontrolled_tags`: free-form editorial/internal tags that exist on recipes but are not valid `tags` filter values.

Tag IDs are German (kebab-style). Pass them verbatim to `search_recipes(tags=[...])`. Multiple tags combine with AND logic — `tags=["hofkueche", "vegetarisch"]` returns only vegetarian dishes that are also tagged as courtly cuisine. Social-class tags can co-occur on a single recipe when a source explicitly addresses multiple classes (classic Bockenheim pattern: *"et erit bonum pro ciuibus Rusticis et nobilibus"* → both `bauernkueche` and `hofkueche`).

The `dietary` filter in `search_recipes` is a convenience alias: `dietary="vegetarian"` is equivalent to `tags=["vegetarisch"]`, and `dietary="vegan"` to `tags=["vegan"]`. Note that vegan recipes also carry the `vegetarisch` tag, so `dietary="vegetarian"` returns both groups.

---

#### `search_recipes`

Search historical recipes with filtering and ingredient matching.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `course` | string | — | See course types below |
| `difficulty_max` | integer 1–3 | — | 1=easy, 2=medium, 3=advanced |
| `lagerkueche` | boolean | — | Only recipes suitable for outdoor/camp cooking |
| `source_key` | string | — | Filter by cookbook key. Call `list_recipe_sources` for the full list (40 active sources). |
| `dietary` | string | — | `vegetarian` (no meat/fish, eggs/dairy allowed) or `vegan` (no animal products; almond milk and honey accepted by convention). Vegan recipes are also tagged vegetarian, so `vegetarian` includes the vegan ones. Equivalent to `tags=["vegetarisch"]` / `tags=["vegan"]`. |
| `tags` | string[] | — | Controlled-vocabulary tag filter (AND logic, max 6). Vocabulary: `pasta`, `reis`, `brei`, `beilage`, `huelsenfruechte`, `brot` (dish type); `vegetarisch`, `vegan`, `fastenspeise` (diet); `hofkueche`, `buergerlich`, `bauernkueche` (social class). Call `list_recipe_tags` for descriptions. |
| `epoch_from` | integer | — | Earliest source year (e.g. `1300`) |
| `epoch_to` | integer | — | Latest source year (e.g. `1500`) |
| `ingredients` | string[] | — | Include filter: all listed must be present (AND). Case-, umlaut- and ß-folded (`Apfel` finds `Äpfel`); bracketed notes such as `Öl (zum Braten der Äpfel)` are ignored; see `ingredient_match`. |
| `ingredient_qids` | string[] | — | Exact filter by Wikidata QID, e.g. `["Q89"]` = apple (AND). Independent of spelling, language and plural. Loses hits for ingredients without a QID (see `qid_coverage`). |
| `ingredient_match` | `word` \| `token` \| `substring` | — | `word` (default): term must start a word of the ingredient name (`Apfel` finds `Äpfel`, `Apfelessig`, not `Granatäpfel`). `token`: term must be a whole word (not `Apfelessig`). `substring`: anywhere (also `Granatäpfel`, `Jakober-Äpfel`). Plain plural stem on the term (`Birnen` = `Birne`). |
| `query` | string | — | Free-text search (min. 3 chars; terms AND-ed, umlaut-folded). Terms ≤ 4 chars must start a word; longer terms match anywhere (`Torte` finds `Quittentorte`, not `Törtchen`). |
| `query_in` | array | — | Where `query` looks: `title`, `text`, `ingredients`, `transcript`. Default `["title","text","ingredients"]`. Scoring per term and field, word start beats inside-a-compound: title 100/60, ingredients 30/15, text 10/5, transcript 3/1; ties by `quality_score`, then `id`. |
| `sort` | `relevance` \| `quality_score` \| `id` | — | Default `relevance` with `query`, else `id`. Relevance: (1) recipes with a title hit first, word-start before inside-a-compound; (2) among recipes without a title hit: ingredient hits before text hits; (3) `quality_score` descending; (4) `id`. Among title hits the order is therefore purely `quality_score`. Every order tie-breaks on `id`, so paging is stable. |
| `exclude_courses` | string[] | — | Exclude these course types |
| `exclude_ingredients` | string[] | — | Exclude recipes containing any of these ingredients (same matching as `ingredients`) |
| `limit` | integer | — | Page size, default 20, max 100 |
| `offset` | integer | — | Skip this many matches; pass `next_offset` from the previous response |
| `detail` | `slim` \| `standard` | — | Default `slim`: id, source_key, source_ref, title_modern, course, lagerküche, difficulty, prep_time_min, fyndling_url. `standard` = full list entry (text, ingredients, annotations, FAQ). |
| `fields` | string[] | — | Whitelisted fields ADDED to the default (e.g. `quality_score`, `tags`, `score`, `matched_ingredients`); missing values come back as `null`/`[]`, unknown names are rejected. |
| `fields_mode` | `add` \| `only` | — | `add` (default) extends the default; `only` returns just `fields` (+ `id`, `fyndling_url`). |

**Response:** `total` (all matches), `count` (this page), `offset`, `sort`, `has_more`, `next_offset`, `recipes`. With an ingredient filter each hit carries `matched_ingredients`; with `ingredient_qids` the response adds `qid_coverage` (only ~75% of ingredient lines have a QID — recipes without one are not found by that filter). Results are ordered by id (title hits first with `query`), so check `has_more` on broad searches.

**Course types:**

| Value | Description |
|---|---|
| `starter` | Starters / appetisers |
| `main_beef` | Beef mains |
| `main_pork` | Pork mains |
| `main_poultry` | Poultry mains (chicken, goose, …) |
| `main_game` | Game mains (venison, hare, …) |
| `main_fish` | Fish mains |
| `main_other` | Other mains |
| `main_vegetarian` | Vegetarian mains |
| `main_side` | Side-dish mains |
| `main_meat` | Meat mains whose animal is unspecified (mutton, lamb, offal, mixed) — **and** the alias for all meat mains, see below |
| `main` | **Alias** — every main course (beef, pork, poultry, game, meat, fish, vegetarian, side, other) |
| `side` | Side dishes |
| `dessert` | Desserts / sweet dishes |
| `drink` / `beverage` | Beverages (`beverage` is an alias for `drink`; `cellar` craft is not a drink) |
| `condiment` | Sauces, spice pastes |
| `other` | Miscellaneous |

`main_meat` is deliberately both a stored value and an alias: unspecified meat is meat, so `course=main_meat` returns every meat main (beef, pork, poultry, game and the unspecified ones) — never vegetarian or fish. `beverage` does not include `cellar`.

**Example — desserts with cinnamon and ginger:**
```json
{
  "course": "dessert",
  "ingredients": ["Zimt", "Ingwer"],
  "limit": 5
}
```

**Example — easy camp-cooking poultry dishes from before 1450:**
```json
{
  "course": "main_poultry",
  "difficulty_max": 1,
  "lagerkueche": true,
  "epoch_to": 1450
}
```

**Example — all recipes from the oldest source (13th-century Denmark):**
```json
{
  "source_key": "harpestreng",
  "limit": 25
}
```

**Example — Flemish court cuisine from the late 15th century:**
```json
{
  "source_key": "edelike_spijse",
  "course": "main_poultry",
  "limit": 10
}
```

**Example — the Regensburg manuscript (Cgm 5919, ~1500):**
```json
{
  "source_key": "muenchner_cgm5919",
  "limit": 20
}
```

**Example — vegan desserts (lent-friendly sweets, no animal products):**
```json
{
  "course": "dessert",
  "dietary": "vegan",
  "limit": 10
}
```

**Example — courtly bread-based dishes (banquet-grade Backwerk):**
```json
{
  "tags": ["hofkueche", "brot"],
  "limit": 10
}
```

**Example — peasant-class lent food (rural fast-day dishes):**
```json
{
  "tags": ["bauernkueche", "fastenspeise"],
  "limit": 20
}
```

**Recipe list fields** (full details stripped for list performance; v2 also drops `applied_delta_ids`, `quality_score_details`, `review_dates` from list results — `get_recipe` still returns everything): `id`, `source_key`, `title_modern`, `course`, `difficulty`, `serves`, `prep_time_min`, `ingredients`, `lagerküche`, `published_at`, `fyndling_url` (canonical link to the recipe page on fyndling.de)

---

#### `get_recipe`

Get the full details of a single recipe.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `id` | string | ✓ | Recipe ID (e.g. `hkb-001`, `bgs-001`, `foc-015`, `men-042`, `m384-001`, `m5919-010`) |

**Full response includes:**
- `text_modern` — modern German adaptation of the recipe
- `ingredients` — structured list with `amount`, `unit`, `name`, `original_text`, `original` (medieval source text)
- `transcript` — original medieval text with language and source
- `annotations` — glossary of archaic terms
- `faq` — common questions answered
- `interpretive_choices` — editorial decisions on ambiguous passages
- `scan` — link to manuscript scan image
- `fyndling_url` — canonical link to the recipe page on fyndling.de (e.g. `https://fyndling.de/rezepte/mar-005/`)

**Ingredient units:** `g`, `kg`, `ml`, `l`, `TL` (Teelöffel/teaspoon), `EL` (Esslöffel/tablespoon), `tsp`, `tbsp`, `pinch`, `piece`, `slice`, `clove`, `bunch`, `sprig`, `leaf`, `cm`. Use `original_text` for display; `amount` + `unit` are for scaling only.

**Example ingredient object:**
```json
{
  "original": "ein phunt mandels",
  "amount": 500,
  "unit": "g",
  "name": "Mandeln",
  "original_text": "500 g Mandeln"
}
```

---

#### `compose_menu`

Compose a multi-course menu from historical recipes, one recipe per course. Per course the pick is: least ingredient overlap with the courses chosen so far, then a cookbook not yet used in the menu (menus mix sources when quality allows), then highest `quality_score`. Candidates are the 200 best-reviewed matches per course.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `courses` | string[] | ✓ | Ordered course list, 1–6 entries (use course type values from above) |
| `persons` | integer | — | Number of persons. **Echo only** — amounts are not scaled. |
| `detail` | `slim` \| `standard` | — | Default `slim` (id, title, course, source, difficulty, prep time, quality_score, URL; ~400 tokens for 3 courses). `standard` returns the full entries (~8k tokens per course). |
| `max_difficulty` | integer 1–3 | — | Maximum difficulty for any course |
| `lagerkueche` | boolean | — | Only camp-cooking-suitable recipes |
| `dietary` | string | — | `vegetarian` or `vegan` — applied to every course (vegan recipes are also tagged vegetarian) |
| `tags` | string[] | — | Controlled-vocabulary tags applied to every course (AND logic, max 6). Useful for thematic menus, e.g. `["hofkueche"]` for a courtly banquet or `["bauernkueche"]` for a peasant-class meal. See `list_recipe_tags` for the vocabulary. |
| `epoch_from` | integer | — | Earliest source year |
| `epoch_to` | integer | — | Latest source year |

**Example — 4-course dinner for 8, 14th-century only:**
```json
{
  "courses": ["starter", "main_fish", "main_poultry", "dessert"],
  "persons": 8,
  "epoch_from": 1300,
  "epoch_to": 1400
}
```

**Example — vegetarian 3-course menu for 6:**
```json
{
  "courses": ["starter", "main_vegetarian", "dessert"],
  "persons": 6,
  "dietary": "vegetarian"
}
```

**Example — courtly banquet (5-course Hofküche dinner for 12):**
```json
{
  "courses": ["starter", "main_fish", "main_poultry", "main_game", "dessert"],
  "persons": 12,
  "tags": ["hofkueche"]
}
```

---

## Coverage

**Events:** Germany, Austria, Switzerland, France, Poland, Czech Republic, Italy, Spain, Portugal, UK, Ireland, Belgium, Netherlands, Luxembourg, Denmark, Sweden, Norway, Finland, Estonia, Lithuania, and more — plus renaissance faires in the United States, Canada, Mexico, and beyond.

**Recipes:** Forty cookbooks spanning Old Danish, Old French, Middle High German, Middle English, Tuscan Volgare, Medieval Latin, Middle Dutch, Early New High German (Bavarian, Alemannic–Swabian, Rhine-Franconian, Ripuarian), Early Italian, and Early Czech — from Copenhagen, Paris, London, Würzburg, Florence, the papal court at Rome, Ghent, Cologne, Solothurn, the Upper Rhine, northern Italy, Prague, Frankfurt-am-Main, and a dense cluster of South-German manuscripts: Munich (BSB Cgm 384, 467, 725, 811, 5919, Cgm 349, Clm 15632), Tegernsee, Rott am Inn, Dietramszell, Regensburg, the Reichenau/Lake Constance region (Karlsruhe, Cod. Aug. pap. 125), the lower Inn valley (Karlsruhe, Cod. Donaueschingen 793), and the Teutonic-Order Königsberg fragment. Covering the 13th to 16th century.

Notable sources: The Harpestreng manuscript (NKS 66, ~1300) is the earliest surviving cookbook from northern Europe. Le Viandier de Taillevent (~1300) is one of the most influential French court cookbooks of the Middle Ages. The Registrum Coquine of Johannes von Bockenheim (~1433, BnF Ms. Latin 7054) is a Latin compilation from the papal court of Martin V that explicitly labels recipes by social class — "pro magnatibus", "pro communibus", "pro rusticis". The Ghent manuscript (BHSL.HS.1035, ~1475) is the only fully preserved Middle Dutch recipe collection of its era. The Tegernseer Speisenbuch (BSB Cgm 8137, 1453–1534) documents Benedictine monastery cuisine from Bavaria and contains the oldest known written record of the name *Rutschart* (today's *Ritschert*). The South-German manuscript cluster (Munich/Regensburg/Tegernsee) makes Fyndling one of the largest structured digital corpora of 15th–16th-century German cookbook recipes.

---

## License & Attribution

Event data is aggregated from public sources; accuracy is not guaranteed — always verify with the organiser.

Recipe texts and modern adaptations: © [Fyndling](https://fyndling.de), CC BY-SA 4.0. Original medieval texts are in the public domain.
</content>
</invoke>
