# vehicles-database

PostgreSQL database `vehicles` — a vehicle reference catalog (brands, models,
generations, modifications, trims, options) plus general-purpose reference data
(Ukraine's administrative divisions, body colors).

## Structure

Two schemas:

- **`public`** — the vehicle catalog.
- **`reference`** — general-purpose reference data: Ukraine geography and colors.

## Tables

### `public` — vehicle catalog

| Table | Rows | Description |
|---|---:|---|
| `brands` | 5,583 | Vehicle brands/makes |
| `categories` | 10 | Vehicle categories (cars, crossovers, trucks, etc.) |
| `brand_categories` | 6,745 | Brand ↔ category link (many-to-many) |
| `models` | 41,359 | Models, linked to brand and category |
| `generations` | 19,436 | Model generations (production years) |
| `modifications` | 128,172 | Generation modifications (engine, gearbox, fuel) |
| `equipments` | 154,236 | Modification trims (by model year) |
| `options` | 131 | Options/equipment |
| `category_options` | 828 | Options available for a category |
| `body_styles` | 326 | Body styles |
| `category_body_styles` | 329 | Body styles available for a category |
| `drive_types` | 6 | Drive types (FWD/RWD/AWD, etc.) |
| `category_drive_types` | 21 | Drive types available for a category |
| `gearboxes` | 6 | Gearbox types |
| `category_gearboxes` | 36 | Gearboxes available for a category |
| `fuels` | 10 | Fuel types |
| `countries` | 57 | Countries (brand's country of origin) |

Plus two ready-made views:

- `v_model_summary` — model + brand + category + generation/modification counts + production years.
- `v_modification_full` — modification with the full chain: generation → model → brand → category + fuel + gearbox.

### `reference` — reference data

| Table | Rows | Description |
|---|---:|---|
| `regions` | 5 | Macro-regions of Ukraine |
| `states` | 23 | Oblasts (`region_id` → `regions`) |
| `cities` | 1,496 | Cities/settlements (`state_id` → `states`) |
| `colors` | 11 | Body colors (name + hex) |

Hierarchy: `regions` → `states` → `cities` (`region_id`, `state_id` — regular FKs within the schema).

## About `regions` / `states` / `cities` (Ukraine)

This is a three-level administrative/geographic reference for Ukraine:

1. **`regions`** (5 rows) — conventional macro-regions grouping the oblasts:
   Center, West, East, South, North. Used for on-the-fly geographic filtering
   (e.g. "show listings from Western Ukraine").
2. **`states`** (23 rows) — oblasts of Ukraine (Kyivska, Lvivska, Odeska, etc.),
   each linked to one macro-region via `region_id`.
3. **`cities`** (1,496 rows) — cities and settlements, each linked to an oblast
   via `state_id`.

## Importing the dump

The dump is at the repository root: `vehicles_dump.sql` (plain SQL, no owner/privileges,
21 tables + 2 views, ~34 MB).

```bash
# 1. Create the database (if it doesn't exist yet)
createdb -U <user> vehicles

# 2. Load the dump
psql -U <user> -d vehicles -f vehicles_dump.sql
```

Or in one line, if the database already exists:

```bash
psql -U <user> -h <host> -p <port> -d vehicles -f vehicles_dump.sql
```

If Postgres runs in a Docker container (e.g. container `postgres`):

```bash
docker exec -i <container_name> psql -U <user> -d vehicles < vehicles_dump.sql
```

## Example queries

### Brands by category

```sql
SELECT b.id, b.name, b.slug, c.iso_alpha2 AS country
FROM brands b
JOIN brand_categories bc ON bc.brand_id = b.id
JOIN countries c ON c.id = b.country_id
WHERE bc.category_id = (SELECT id FROM categories WHERE slug = 'legkovi')  -- Cars
  AND b.is_active = true
ORDER BY b.name;
```

### Models by category and brand

```sql
SELECT m.id, m.name, m.slug
FROM models m
WHERE m.brand_id = (SELECT id FROM brands WHERE slug = 'toyota')
  AND m.category_id = (SELECT id FROM categories WHERE slug = 'legkovi')
  AND m.is_active = true
ORDER BY m.name;
```

### Generations for a model

```sql
SELECT g.id, g.name, g.year_from, g.year_to
FROM generations g
WHERE g.model_id = (SELECT id FROM models WHERE slug = 'camry' AND brand_id = (SELECT id FROM brands WHERE slug = 'toyota'))
  AND g.is_active = true
ORDER BY g.year_from DESC;
```

### Modifications for a generation

```sql
SELECT mod.id, mod.name, mod.engine_volume, mod.engine_power_hp,
       f.name AS fuel, gb.name AS gearbox
FROM modifications mod
LEFT JOIN fuels f ON f.id = mod.fuel_id
LEFT JOIN gearboxes gb ON gb.id = mod.gearbox_id
WHERE mod.generation_id = 12345
  AND mod.is_active = true
ORDER BY mod.engine_power_hp DESC;
```

### Trims for a modification

```sql
SELECT e.id, e.name, e.model_year
FROM equipments e
WHERE e.modification_id = 67890
  AND e.is_active = true
ORDER BY e.model_year DESC;
```

### Drive types by category

```sql
SELECT dt.id, dt.name, dt.code
FROM drive_types dt
JOIN category_drive_types cdt ON cdt.drive_type_id = dt.id
WHERE cdt.category_id = (SELECT id FROM categories WHERE slug = 'legkovi')  -- Cars
ORDER BY dt.name;
```

### Options by category

```sql
SELECT o.id, o.name, o.option_group
FROM options o
JOIN category_options co ON co.option_id = o.id
WHERE co.category_id = (SELECT id FROM categories WHERE slug = 'legkovi')
ORDER BY o.option_group, o.name;
```

### Combined queries

**Full hierarchy brand → model → generation → modification** (via the ready-made view):

```sql
SELECT brand_name, model_name, generation_name, year_from, year_to,
       modification_name, engine_volume, engine_power_hp, fuel_name, gearbox_name
FROM v_modification_full
WHERE brand_slug = 'toyota' AND model_slug = 'camry'
ORDER BY year_from DESC, engine_power_hp DESC;
```

**Model summary with body styles and generation/modification counts:**

```sql
SELECT * FROM v_model_summary
WHERE brand_slug = 'bmw'
ORDER BY generation_count DESC;
```

**Body styles + gearboxes + drive types available for a category (full category profile):**

```sql
SELECT
    (SELECT array_agg(bs.name) FROM category_body_styles cbs JOIN body_styles bs ON bs.id = cbs.body_style_id WHERE cbs.category_id = c.id) AS body_styles,
    (SELECT array_agg(gb.name) FROM category_gearboxes cg JOIN gearboxes gb ON gb.id = cg.gearbox_id WHERE cg.category_id = c.id) AS gearboxes,
    (SELECT array_agg(dt.name) FROM category_drive_types cdt JOIN drive_types dt ON dt.id = cdt.drive_type_id WHERE cdt.category_id = c.id) AS drive_types
FROM categories c
WHERE c.slug = 'legkovi';
```

**Top 10 brands by number of active models:**

```sql
SELECT b.name, count(*) AS models_count
FROM models m
JOIN brands b ON b.id = m.brand_id
WHERE m.is_active = true
GROUP BY b.name
ORDER BY models_count DESC
LIMIT 10;
```

**Fuzzy search for brand/model by name** (uses the `pg_trgm` indexes
`idx_brands_name_trgm`, `idx_models_name_trgm`):

```sql
SELECT name, similarity(name, 'toyota') AS score
FROM brands
WHERE name % 'toyota'
ORDER BY score DESC
LIMIT 5;
```

**Listings (hypothetical example) joined with the `reference` schema: vehicles of a given color in a given oblast:**

```sql
SELECT s.name AS state, col.name AS color, count(*) AS listings_count
FROM listings l                                  -- table from another database/service
JOIN reference.cities c ON c.id = l.city_id
JOIN reference.states s ON s.id = c.state_id
JOIN reference.colors col ON col.id = l.color_id
GROUP BY s.name, col.name
ORDER BY listings_count DESC;
```

**Cities in a given oblast:**

```sql
SELECT c.name
FROM reference.cities c
JOIN reference.states s ON s.id = c.state_id
WHERE s.name = 'Lvivska'
ORDER BY c.name;
```
