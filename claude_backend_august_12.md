# Prescription Frames — Build Log (August 12, 2026)

August 2026. Full pipeline from raw product pages to a live API endpoint. 86 frames across 4 brands.

---

## What Was Built

| Piece | Description |
|---|---|
| `prescription-frames.json` | Curated catalog of 86 prescription-compatible frames |
| `validate.py` | Python validation script — 16 checks across every field |
| `migrate.py` | Uploads images to Supabase storage, upserts rows into DB |
| `005_prescription_frames.sql` | DB migration — new `prescription_frames` table |
| `GET /api/public/prescriptions` | Public endpoint serving the catalog |

---

## Data Pipeline

### 1. Manual data entry (product pages → JSON)

No scraper. Went through each brand's product page one by one:

- Took a screenshot of the product
- Renamed the screenshot to `<slug>.png` and dropped it in `prescription-frames/images/`
- Appended a JSON entry to `prescription-frames.json`

This was deliberate — automated scrapers would miss prescription-specific details (rx range, which variants are available for Rx) that required human judgment.

### 2. JSON schema

```json
{
  "name": "Wiley X Gravity",
  "slug": "wiley-x-gravity",
  "image": "images/wiley-x-gravity.png",
  "priceCents": 18900,
  "size": "SM-MED",
  "rxLow": -8.0,
  "rxHigh": 2.0,
  "colors": [
    { "option": "Matte Black Frame", "slug": "matte-black-frame", "value": "#2b2b2b" }
  ]
}
```

### 3. Naming conventions

**Slug** — matches the slugify function in `src/lib/utils.ts`:
```python
def slugify(name):
    s = name.lower().strip()
    s = re.sub(r"[^a-z0-9]+", "-", s)
    s = re.sub(r"^-+|-+$", "", s)
    return s
```

**Name cleaning** — strip "for Rx", "for High Rx", "for High Power Rx", and any bracket content from the product page title.

**High power variants** — append "High Power" suffix: `Wiley X Gravity High Power`.

**Size tokens** — always one of: `XS`, `SM`, `MED`, `LG`, `XL`, `XXL`. Ranges use a dash: `SM-MED`. Non-standard tokens get normalized: `XLG→XL`, `XSM→XS`, `MD→MED`.

**Ziena brand** — eyecup color combos simplified to frame color only (eyecup is an accessory, not a frame variant).

**Rx range rule** — never assume. If the product page doesn't state a prescription range, flag it and skip. Two entries (Ziena Marina, Ziena Seacrest) were removed for this reason.

---

## Validation Script (`validate.py`)

Written in Python. Runs against the local JSON file before migration.

### 16 checks

1. `name` is present
2. `slug` is unique (uses a set — adding a duplicate to a set in Python is a no-op, so we check membership first)
3. `slug` matches `slugify(name)`
4. `image` field equals `images/<slug>.png`
5. Image file actually exists on disk
6. `priceCents` is present
7. `priceCents > 3500` (guards against entering dollars instead of cents, e.g. 129 instead of 12900)
8. `size` is present
9. All size tokens are valid (`XS`, `SM`, `MED`, `LG`, `XL`, `XXL`)
10. Size tokens are in ascending order
11. `rxLow` is present
12. `rxHigh` is present
13. `rxLow < rxHigh`
14. `colors` is non-empty
15. Each color's `slug` matches `slugify(color["option"])`
16. Each color's `value` is a valid lowercase hex (`#[0-9a-f]{6}`)

### Key Python patterns used

```python
# Script-relative paths (works regardless of where you run from)
JSON_FILE = os.path.join(os.path.dirname(__file__), "prescription-frames.json")

# Reading JSON from a file object
with open(JSON_FILE) as f:
    frames = json.load(f)   # json.load takes a file; json.loads takes a string

# Unique slug check
seen_slugs = set()
if slug in seen_slugs:     # check before adding
    error()
seen_slugs.add(slug)       # adding a duplicate is a no-op, doesn't raise

# Keyed dict access vs .get()
color["option"]            # raises KeyError if missing — intentional, catches bad data
color.get("option", "")    # returns "" silently — masks bugs

# Hex color validation
re.fullmatch(r"#[0-9a-f]{6}", value)   # fullmatch requires the entire string to match

# Non-zero exit on failure
sys.exit(1)
```

---

## Database

### Table (`005_prescription_frames.sql`)

```sql
create table prescription_frames (
  id          uuid        primary key default gen_random_uuid(),
  brand_slug  text        not null references brands(slug) on delete cascade,
  name        text        not null,
  slug        text        not null unique,
  image_src   text        not null,
  price_cents int         not null,
  size        text        not null,
  rx_low      numeric     not null,
  rx_high     numeric     not null,
  colors      jsonb       not null,
  created_at  timestamptz not null default now(),
  updated_at  timestamptz not null default now()
);

alter table prescription_frames enable row level security;
```

No RLS policy — the public endpoint uses the admin client which bypasses RLS. A `select` policy for `anon` would be dead code since we never use the anon client for this table.

---

## Migration Script (`migrate.py`)

### What it does

For each frame in `prescription-frames.json`:
1. Uploads `images/<slug>.png` to `bikershades/prescriptions/<slug>.png` in Supabase storage (upsert — safe to re-run)
2. Constructs the public `image_src` URL
3. Upserts the row into `prescription_frames` on conflict with `slug`

### Idempotent by design

Running the script multiple times is safe — storage upload uses `upsert: true`, DB insert uses `on_conflict="slug"`. Re-running after changing `STORAGE_FOLDER` from `prescription` to `prescriptions` updated all `image_src` URLs in the DB, leaving the old folder orphaned in storage (manually deleted from Supabase UI).

### Key patterns

```python
# Load .env.local manually (no dotenv library)
def load_env():
    env = {}
    with open(ENV_FILE) as f:
        for line in f:
            line = line.strip()
            if line and not line.startswith("#") and "=" in line:
                key, _, value = line.partition("=")  # partition is safer than split("=", 1)
                env[key.strip()] = value.strip()
    return env

# Path relative to script file, not cwd
ENV_FILE = Path(__file__).parent.parent / ".env.local"

# Upsert with conflict target
supabase.table("prescription_frames").upsert({...}, on_conflict="slug").execute()

# Error counting — script continues on failure, reports at end
try:
    ...
except Exception as e:
    print(f"[{slug}] ERROR: {e}")
    errors += 1
```

### Running it

```bash
python3 prescription-frames/migrate.py
```

Result: `86/86 migrated.`

---

## API Endpoint

`GET /api/public/prescriptions?brandSlug=bikershades`

```typescript
// src/app/api/public/prescriptions/route.ts
export async function GET(req: NextRequest) {
  const brandSlug = req.nextUrl.searchParams.get("brandSlug");
  if (!brandSlug) return err("brandSlug is required", 400);

  const { data, error } = await supabase
    .from("prescription_frames")
    .select("id, name, slug, image_src, price_cents, size, rx_low, rx_high, colors")
    .eq("brand_slug", brandSlug)
    .order("name");

  // rx_low / rx_high come back as strings from Postgres numeric — Number() converts them
  const frames = (data ?? []).map((f) => ({
    ...
    rxLow: Number(f.rx_low),
    rxHigh: Number(f.rx_high),
  }));

  return ok(frames);
}
```

Note: Supabase returns `numeric` columns as strings to preserve precision. `Number()` converts them to JS floats on the response.

---

## Brands Covered

| Brand | Frames |
|---|---|
| BikerArmour | 30 |
| 7eye | 22 |
| Wiley X | 28 (inc. high power variants) |
| Ziena | 1 (Nereus) |

**Total: 86 frames**

---

## macOS Side Note

When dragging a folder in Finder between locations on the same volume, macOS creates an **alias** (a pointer back to the original) rather than copying the data. Clicking the alias navigates to the original. Use copy-paste (`⌘C` / `⌘V`) to get an actual duplicate.

Files downloaded from the internet carry a quarantine extended attribute, shown as `@` in `ls -l` output. Harmless, can be stripped with `xattr -cr <path>`.
