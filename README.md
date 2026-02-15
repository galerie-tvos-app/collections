# ArtSaver Collection Specification (v2.0)

This specification defines the structure for art collections. Both bundled and remote GitHub collections support **TOML** (preferred) and **JSON** (paradata format). When both formats exist for the same collection, TOML takes precedence.

A *collection* may represent:

- A broad category (e.g., "Masters", "Impressionism", "Landscapes")
- A curated playlist
- A museum-specific selection
- A user-created collection

A *sub-collection* groups artworks within a collection:

- An artist (e.g., "Van Gogh", "Renoir")
- A style or period (e.g., "Dutch Golden Age")
- A gallery room or mood

Collections may include artworks from **any museum source**.

---

# 1. TOML Format (Bundled Collections)

TOML is the preferred format for both bundled and remote collections. Items are minimal -- just `museum` + `object` -- and metadata is auto-resolved from museum APIs at runtime.

## 1.1 Collection Header

```toml
[collection]
name = "Masters"
description = "A cross-museum celebration of humanity's most influential artists"
```

## 1.2 Sub-Collections

```toml
[sub.van-gogh]
name = "Vincent van Gogh"
description = "Intensity, color, and emotion"
default_artist = "Vincent van Gogh"
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | yes | Display name |
| `description` | string | yes | Short description |
| `default_artist` | string | no | Inherited by items that don't specify `artist` |

The sub-collection key (e.g., `van-gogh`) becomes the sub-collection ID.

## 1.3 Items

```toml
[[sub.van-gogh.items]]
museum = "met"
object = "436121"
```

That's it. Two fields. The app derives:

- **`id`** as `{museum}-{object}` (e.g., `met-436121`)
- **`artist`** from `default_artist` on the sub-collection
- **`title`, `date`, `medium`, `classification`, `subjects`** from the museum API (cached after first fetch)
- **`imageURL`, `thumbnailURL`** resolved at runtime (never in manifest)

### Optional Per-Item Overrides

Any metadata field can be overridden per item. TOML overrides take precedence over API-resolved values:

```toml
[[sub.van-gogh.items]]
museum = "met"
object = "436839"
artist = "John Singer Sargent"    # override default_artist

[[sub.van-gogh.items]]
museum = "nga"
object = "54ee6643-e0f9-4b92-a1d2-441e5108724d"
title = "Self-Portrait"           # NGA needs manual metadata (see Section 2.4)
date = "1889"
medium = "Oil on canvas"
classification = "Painting"
subjects = ["Portrait"]
style = "Post-Impressionism"
focus_y = 30                      # face is in the upper third
```

### Item Fields Reference

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `museum` | string | yes | Museum identifier (see Section 5) |
| `object` | string | yes | Museum's object ID (IIIF UUID for NGA -- see Section 2.4) |
| `title` | string | no | Override API-resolved title |
| `artist` | string | no | Override `default_artist` and API-resolved artist |
| `date` | string | no | Override API-resolved date |
| `medium` | string | no | Override API-resolved medium |
| `classification` | string | no | Override API-resolved classification |
| `subjects` | [string] | no | Override API-resolved subjects |
| `style` | string | no | Art style tag (e.g., `"Impressionism"`, `"Baroque"`) for classification |
| `focus_y` | integer | no | Vertical focus percentage (0-100). When cropping for landscape display, the crop centers on this point. 0 = top, 50 = center (default), 100 = bottom. |

## 1.4 File Naming

TOML collection files are named `{collectionId}.toml`:
- `masters.toml`
- `impressionism.toml`
- `landscapes.toml`

## 1.5 Complete TOML Example

```toml
[collection]
name = "Impressionism"
description = "Light, color, and atmosphere"

[sub.monet]
name = "Claude Monet"
description = "The poetry of perception"
default_artist = "Claude Monet"

[[sub.monet.items]]
museum = "met"
object = "437853"

[[sub.monet.items]]
museum = "nga"
object = "99758d9d-c10b-4d02-a198-7e49afb1f3a6"
title = "Woman with a Parasol"
date = "1875"
medium = "Oil on canvas"
classification = "Painting"
subjects = ["Portrait", "Landscape"]

[sub.paris-was-a-mood]
name = "Paris Was a Mood"
description = "Cabarets, cafes, and nightlife"

[[sub.paris-was-a-mood.items]]
museum = "aic"
object = "28560"

[[sub.paris-was-a-mood.items]]
museum = "aic"
object = "61057"
```

---

# 2. Metadata Resolution

## 2.1 How It Works

```
1. TOML parsed -> minimal items (museum + object + optional overrides)
2. For items missing metadata:
   a. Check metadata cache (Documents/metadata_cache.json)
   b. If not cached, call museum API
   c. Extract title/artist/date/medium/classification/subjects
   d. Cache result to disk
   e. Merge: TOML overrides > API response > default_artist
3. URL resolution happens separately (as before)
```

The metadata cache persists across launches. Each item's API is called once, ever.

## 2.2 What Each Museum API Provides

| Field | Met | AIC | NGA | Wikimedia |
|-------|-----|-----|-----|-----------|
| title | `title` | `data.title` | open data* | -- |
| artist | `artistDisplayName` | `data.artist_title` | open data* | -- |
| date | `objectDate` | `data.date_display` | open data* | -- |
| medium | `medium` | `data.medium_display` | open data* | -- |
| classification | `classification` | `data.classification_title` | open data* | -- |
| subjects | `tags[].term` | `data.subject_titles` | -- | -- |

**Met** and **AIC** have live object APIs -- metadata auto-resolves at runtime.

**NGA** has metadata via its [Open Data Program](https://github.com/NationalGalleryOfArt/opendata) (CSV/JSON exports keyed by numeric `objectid`), but no live query API suitable for runtime resolution. Items currently need manual metadata in the TOML file.

**Wikimedia** has no structured metadata API -- items need manual metadata.

## 2.3 Minimum Required Metadata Per Museum

| Museum | In TOML | Auto-Resolved |
|--------|---------|---------------|
| **Met** | `museum` + `object` | title, artist, date, medium, classification, subjects |
| **AIC** | `museum` + `object` | title, artist, date, medium, classification, subjects |
| **NGA** | `museum` + `object` (IIIF UUID) + metadata fields | URLs only (IIIF direct from UUID) |
| **Wikimedia** | `museum` + `object` + metadata fields | URLs only |
| **Other** | `museum` + `object` + metadata fields | Nothing (needs `imageURL` in JSON) |

## 2.4 NGA's Two-ID System

NGA uses two different identifiers:

- **`objectid`** (numeric, e.g., `46089`) -- used on the NGA website (`nga.gov/artworks/46089-...`) and in the open data CSVs. Contains metadata (title, artist, date, medium, classification).
- **IIIF UUID** (e.g., `132e789c-27da-4f54-a68d-3e2b691a9448`) -- used for image URLs (`api.nga.gov/iiif/{uuid}/full/...`). Found in the `published_images` open data, linked to the objectid via `depictstmsobjectid`.

Currently, the `object` field in TOML stores the **IIIF UUID** so the app can construct image URLs directly (no API call needed). The trade-off is that metadata must be provided manually in the TOML.

**Finding NGA IDs:**
1. Find the artwork on [nga.gov](https://www.nga.gov/) -- the numeric objectid is in the URL (e.g., `nga.gov/artworks/46089-...`)
2. Look up the IIIF UUID in [published_images.csv](https://github.com/NationalGalleryOfArt/opendata/blob/main/data/published_images.csv) by matching `depictstmsobjectid`
3. Look up metadata in [objects.csv](https://github.com/NationalGalleryOfArt/opendata/blob/main/data/objects.csv) by `objectid`

**Future:** Bundle a trimmed NGA lookup table (objectid -> UUID + metadata) so items can use just the numeric objectid and auto-resolve everything.

---

# 3. JSON Format (Legacy/Fallback)

JSON (`.paradata.json`) is the fallback format when no `.toml` file exists. Supported for both bundled and remote collections.

## 3.1 Collection Object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `collectionId` | string | yes | Unique ID (e.g., `"masters"`) |
| `collectionName` | string | yes | Display name |
| `description` | string | yes | Short description |
| `version` | string | yes | Version string (e.g., `"2026.02.07"`) |
| `lastUpdated` | string | yes | ISO-8601 timestamp |
| `subThemes` | array | yes | Array of sub-collection objects |

**Important:** The JSON key is `subThemes` (not `subCollections`).

## 3.2 Sub-Collection Object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `subThemeId` | string | yes | Unique within this collection |
| `subThemeName` | string | yes | Display name |
| `description` | string | yes | Short description |
| `items` | array | yes | Array of artwork objects |

## 3.3 Artwork Object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | yes | Unique ID (e.g., `"met-436535"`) |
| `museumId` | string | yes | Museum identifier (see Section 5) |
| `sourceObjectId` | string | yes* | Museum's object ID |
| `title` | string | yes | Artwork title |
| `artist` | string | yes | Artist name |
| `date` | string | yes | Date or date range |
| `medium` | string | yes | Medium description |
| `classification` | string | yes | Type (e.g., `"Painting"`) |
| `subjects` | [string] | yes | Subject tags |
| `style` | string/null | no | Art style tag (e.g., `"Impressionism"`) |
| `focusY` | integer/null | no | Vertical focus percentage (0-100) for landscape cropping |
| `imageURL` | string/null | no | Pre-resolved image URL |
| `thumbnailURL` | string/null | no | Pre-resolved thumbnail URL |
| `license` | string/null | no | License (e.g., `"CC0"`) |
| `source` | string/null | no | Source attribution |

\* Required for image resolution. Without it, the app cannot resolve image URLs.

## 3.4 File Naming

JSON collection files are named `{collectionId}.paradata.json`:
- `masters.paradata.json`
- `impressionism.paradata.json`

## 3.5 JSON Example

```json
{
  "collectionId": "impressionism",
  "collectionName": "Impressionism",
  "description": "French Impressionist masterworks",
  "version": "2026.02.07",
  "lastUpdated": "2026-02-07T00:00:00Z",
  "subThemes": [
    {
      "subThemeId": "monet",
      "subThemeName": "Claude Monet",
      "description": "Founder of French Impressionism",
      "items": [
        {
          "id": "met-437127",
          "museumId": "met",
          "sourceObjectId": "437127",
          "title": "Water Lilies",
          "artist": "Claude Monet",
          "date": "1906",
          "medium": "Oil on canvas",
          "classification": "Painting",
          "subjects": ["Landscape", "Nature"],
          "imageURL": null,
          "thumbnailURL": null,
          "license": "CC0",
          "source": "The Metropolitan Museum of Art"
        }
      ]
    }
  ]
}
```

---

# 4. What's NOT in Collection Files

These fields belong in the **Museum API Manifest** (`museum_apis.json`), not in collection files:

- `apiEndpoint`, `apiType`, `imageStrategy`
- `imageField`, `thumbnailField`
- `iiifTemplate`, `iiifBaseUrl`
- `requiresApiKey`
- `titleField`, `artistField`, `dateField`, etc. (metadata field mappings)

The manifest handles all API logic and can be updated independently of collection data.

---

# 5. Supported Museum Sources

| `museumId` | Museum | `object` format | Image Resolution | Metadata Resolution | License |
|-----------|--------|-----------------|------------------|---------------------|---------|
| `met` | The Metropolitan Museum of Art | Numeric (e.g., `"436535"`) | API call | API call | CC0 |
| `aic` | Art Institute of Chicago | Numeric (e.g., `"27992"`) | API + IIIF template | API call | CC0 |
| `nga` | National Gallery of Art | IIIF UUID (e.g., `"132e789c-..."`) | IIIF direct from UUID | Manual (see 2.4) | CC0 |
| `wikimedia` | Wikimedia Commons | Filename without `File:` (e.g., `"Mona_Lisa.jpg"`) | MediaWiki API | Manual in TOML/JSON | Various |

**Finding object IDs:**
- **Met:** The number in the URL: `metmuseum.org/art/collection/search/436535` -> `"436535"`
- **AIC:** The number in the URL: `artic.edu/artworks/27992` -> `"27992"`
- **NGA:** See Section 2.4. The IIIF UUID must be looked up from the [published_images](https://github.com/NationalGalleryOfArt/opendata/blob/main/data/published_images.csv) open data. The numeric objectid in the NGA URL (`nga.gov/artworks/46089-...`) is **not** what goes in the `object` field.
- **Wikimedia:** The filename from the Commons URL: `commons.wikimedia.org/wiki/File:Starry_Night.jpg` -> `"Starry_Night.jpg"`

---

# 6. TOML vs JSON Comparison

| | TOML (preferred) | JSON (fallback) |
|---|---|---|
| **Adding an item** | 2 lines (`museum` + `object`) | ~15 lines (all metadata required) |
| **Metadata** | Auto-resolved from API | Must be provided manually |
| **Syntax errors** | Rare (no trailing commas, no quotes on keys) | Common (trailing commas, missing quotes) |
| **Thumbnails** | Resolved at runtime | Resolved at runtime |
| **Used for** | Bundled and remote collections | Bundled and remote collections |
| **Precedence** | Wins when both formats exist | Skipped if matching `.toml` exists (by filename, not `collectionId`) |

**Dedup note:** When both `foo.toml` and `foo.paradata.json` exist, dedup is based on the **filename** (stripping the extension), not the internal `collectionId`. For example, if the repo has both `impressionism.toml` and `impressionism.paradata.json`, the JSON is skipped — even if its `collectionId` is different (e.g., `"impressionism-deprecated"`). To keep both, use distinct base filenames (e.g., `impressionism-deprecated.paradata.json`).
