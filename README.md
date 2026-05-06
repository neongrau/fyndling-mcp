# fyndling-mcp

[![smithery badge](https://smithery.ai/badge/ralf/fyndling)](https://smithery.ai/servers/ralf/fyndling) [![neongrau/fyndling-mcp MCP server](https://glama.ai/mcp/servers/neongrau/fyndling-mcp/badges/score.svg)](https://glama.ai/mcp/servers/neongrau/fyndling-mcp)

Built for **medieval market fans, reenactors, and living-history enthusiasts** — and the AI assistants that help them plan. Fyndling MCP gives AI clients direct access to two niche European datasets:

- **Medieval events** — query 2,000+ markets, concerts, castle experiences, and living-history events across 20 European countries by **location + radius + date range** (updated weekly)
- **Permanent POIs** — meaderies, mead producers, castles, and medieval restaurants, also searchable by geo-radius
- **Historical recipes** — 1,100+ recipes from six cookbooks spanning the 13th–17th century, with modern German adaptations, structured ingredient lists, and original manuscript transcripts

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

#### `find_events_near`

Find medieval events near a geographic coordinate, sorted by distance.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `lat` | number | ✓ | Latitude |
| `lon` | number | ✓ | Longitude |
| `radius_km` | number | — | Search radius in km (default 50, max 500) |
| `date_from` | string | — | ISO 8601 start date, e.g. `2026-06-01` |
| `date_to` | string | — | ISO 8601 end date, e.g. `2026-06-30` |
| `types` | array | — | `market`, `concert`, `burg_event`, `living_history`, `renfaire` |
| `limit` | integer | — | Max results (default 20, max 100) |

**Example — markets within 80 km of Vienna this summer:**
```json
{
  "lat": 48.2082, "lon": 16.3738,
  "radius_km": 80,
  "date_from": "2026-06-01", "date_to": "2026-08-31",
  "types": ["market"]
}
```

**Response fields:** `id`, `name`, `date_from`, `date_to`, `city`, `country`, `lat`, `lon`, `distance_km`, `category`, `description`, `fyndling_url`

---

#### `list_events`

List events filtered by category, country, and/or date range.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `category` | string | — | `market`, `concert`, `burg_event`, `living_history`, `renfaire` |
| `country` | string | — | ISO 3166-1 alpha-2 code (e.g. `DE`, `AT`, `FR`, `PL`) |
| `date_from` | string | — | ISO 8601 |
| `date_to` | string | — | ISO 8601 |
| `limit` | integer | — | Default 20, max 100 |

---

#### `get_event`

Get full details for a single event by ID.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `id` | string | ✓ | Event ID (16-char hex, e.g. `a1b2c3d4e5f6a7b8`) |

---

#### `find_pois_near`

Find permanent medieval-themed locations (meaderies, castles, restaurants).

| Parameter | Type | Required | Description |
|---|---|---|---|
| `lat` | number | ✓ | Latitude |
| `lon` | number | ✓ | Longitude |
| `radius_km` | number | — | Default 100, max 1000 |
| `poi_type` | string | — | `meadery`, `metkellerei`, `burg`, `ma_gastronomie` |
| `limit` | integer | — | Default 20, max 100 |

---

### Historical Recipes

#### `list_recipe_sources`

List all six available cookbooks with metadata (year, language, region, recipe count).

No parameters.

**Sources:**

| Key | Title | Year | Language | Recipes |
|---|---|---|---|---|
| `buch-guter-speise` | Das Buch von guter Speise | 1350 | Middle High German | 96 |
| `form-of-cury` | The Forme of Cury | 1390 | Middle English | 192 |
| `menagier` | Ménagier de Paris | 1393 | Old French | 380 |
| `martino` | Libro de Arte Coquinaria | 1465 | Early Italian | 268 |
| `severin` | Kuchařství (Böhmisches Kochbuch) | 1535 | Early Czech | ~100 |
| `koch_kellermeisterei` | Koch und Kellermeisterei | 1574 | Early New High German | 110 |

---

#### `search_recipes`

Search historical recipes with filtering and ingredient matching.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `course` | string | — | See course types below |
| `difficulty_max` | integer 1–3 | — | 1=easy, 2=medium, 3=advanced |
| `lagerkueche` | boolean | — | Only recipes suitable for outdoor/camp cooking |
| `source_key` | string | — | Filter by cookbook (see keys above) |
| `epoch_from` | integer | — | Earliest source year (e.g. `1350`) |
| `epoch_to` | integer | — | Latest source year (e.g. `1500`) |
| `ingredients` | string[] | — | Include filter: all listed must be present (partial match, AND logic) |
| `exclude_courses` | string[] | — | Exclude these course types |
| `exclude_ingredients` | string[] | — | Exclude recipes containing any of these ingredients |
| `limit` | integer | — | Default 20, max 100 |

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
| `main_meat` | **Alias** — all meat mains combined |
| `side` | Side dishes |
| `dessert` | Desserts / sweet dishes |
| `drink` / `beverage` | Beverages (`beverage` is an alias for `drink`) |
| `condiment` | Sauces, spice pastes |
| `other` | Miscellaneous |

**Example — desserts with cinnamon and ginger, excluding Ingwer-heavy recipes:**
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

**Recipe list fields** (full details stripped for list performance): `id`, `source_key`, `title_modern`, `course`, `difficulty`, `serves`, `prep_time_min`, `ingredients`, `lagerküche`, `published_at`

---

#### `get_recipe`

Get the full details of a single recipe.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `id` | string | ✓ | Recipe ID (e.g. `bgs-001`, `foc-015`, `men-042`) |

**Full response includes:**
- `text_modern` — modern German adaptation of the recipe
- `ingredients` — structured list with `amount`, `unit`, `name`, `original_text`, `original` (medieval source text)
- `transcript` — original medieval text with language and source
- `annotations` — glossary of archaic terms
- `faq` — common questions answered
- `interpretive_choices` — editorial decisions on ambiguous passages
- `scan` — link to manuscript scan image

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

Compose a multi-course menu from historical recipes. Automatically minimises ingredient overlap between courses.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `courses` | string[] | ✓ | Ordered course list, 1–6 entries (use course type values from above) |
| `persons` | integer | — | Number of persons (informational, included in output) |
| `max_difficulty` | integer 1–3 | — | Maximum difficulty for any course |
| `lagerkueche` | boolean | — | Only camp-cooking-suitable recipes |
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

---

## Coverage

**Events:** Germany, Austria, Switzerland, France, Poland, Czech Republic, Italy, Spain, Portugal, UK, Ireland, Belgium, Netherlands, Denmark, Sweden, Norway, Estonia, Lithuania, and more.

**Recipes:** Six cookbooks from Würzburg, Paris, London, northern Italy, Prague, and Frankfurt — spanning Middle High German, Middle English, Old French, Early Italian, Early Czech, and Early New High German.

---

## License & Attribution

Event data is aggregated from public sources; accuracy is not guaranteed — always verify with the organiser.

Recipe texts and modern adaptations: © [Fyndling](https://fyndling.de), CC BY-SA 4.0. Original medieval texts are in the public domain.