# ArtSaver Paradata JSON Specification (v1.03)
(API-First, Museum-Agnostic, With Embedded Thumbnails)

This specification defines the structure for art collections, sub-collections, and artworks.
It is designed for a multi-museum, API-resolved image system with stable embedded thumbnails.

A *collection* may represent:

- A broad category (e.g., "Masters", "Impressionism", "Landscapes")
- A curated playlist
- A museum
- A user-created collection

A *sub-collection* (called `subTheme` in the JSON keys) groups artworks within a collection:

- An artist (e.g., "Van Gogh", "Renoir")
- A style or period (e.g., "Dutch Golden Age")
- A gallery room or wing

Collections may include artworks from **any source**, including multiple museums.

---

# 1. Collection Object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `collectionId` | string | yes | Unique ID (e.g., `"masters"`, `"impressionism"`) |
| `collectionName` | string | yes | Display name |
| `description` | string | yes | Short description shown in the UI |
| `version` | string | yes | Semver string (e.g., `"1.0.0"`) |
| `lastUpdated` | string | yes | ISO-8601 timestamp |
| `subThemes` | array | yes | Array of sub-collection objects |

```json
{
  "collectionId": "masters",
  "collectionName": "Old Masters",
  "description": "Masterworks from the great European painters",
  "version": "1.0.0",
  "lastUpdated": "2025-06-15T00:00:00Z",
  "subThemes": [ ... ]
}
```

**Important:** The JSON key is `subThemes` (not `subCollections`). This must be exact.

---

# 2. Sub-Collection Object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `subThemeId` | string | yes | Unique within this collection (e.g., `"van-gogh"`, `"renoir"`) |
| `subThemeName` | string | yes | Display name |
| `description` | string | yes | Short description |
| `items` | array | yes | Array of artwork objects |

```json
{
  "subThemeId": "van-gogh",
  "subThemeName": "Vincent van Gogh",
  "description": "Post-Impressionist works by Van Gogh",
  "items": [ ... ]
}
```

**Important:** `subThemeId` must be unique within its parent collection. Two different collections can have the same `subThemeId` (e.g., both "masters" and "impressionism" can have a `"renoir"` sub-collection).

---

# 3. Artwork Object

Each artwork entry has four groups of fields:

## A. Identification (Required)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | yes | Globally unique ID (e.g., `"met-436535"`, `"wikimedia-starry-night"`) |
| `museumId` | string | yes | Source museum identifier (see Section 8) |
| `sourceObjectId` | string | yes* | The museum's own ID for this artwork (see Section 8) |

\* `sourceObjectId` is technically optional in the Swift model but **required for image resolution**. Without it, the app cannot resolve image URLs from the museum API.

## B. Artwork Metadata (Required)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `title` | string | yes | Artwork title |
| `artist` | string | yes | Artist name |
| `date` | string | yes | Date or date range (e.g., `"1889"`, `"ca. 1665"`) |
| `medium` | string | yes | Medium (e.g., `"Oil on canvas"`) |
| `classification` | string | yes | Type (e.g., `"Painting"`, `"Photograph"`, `"Sculpture"`) |
| `subjects` | [string] | yes | Subject tags (e.g., `["Landscape"]`, `["Portrait", "Mythology"]`) |

## C. Image URLs (Optional)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `imageURL` | string or null | no | Full-resolution image URL |
| `thumbnailURL` | string or null | no | Thumbnail image URL |

These can be set to `null` or omitted entirely. The app resolves them at runtime via the museum API using `museumId` + `sourceObjectId`. If provided, the app will use them directly and skip API resolution for this artwork.

## D. Embedded Thumbnail (Optional but Recommended)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `embeddedThumbnail` | string or null | no | Base64-encoded thumbnail with optional data URI prefix |
| `embeddedThumbnailFormat` | string or null | no | MIME type (e.g., `"image/webp"`) |

Provides instant UI rendering without waiting for API resolution. The app strips the `data:image/webp;base64,` prefix if present, or accepts raw base64.

Use the placeholder `"BASE64_WEBP_THUMBNAIL"` if you don't have a real thumbnail yet — the app ignores this placeholder value.

## E. Provenance (Optional)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `license` | string or null | no | License (e.g., `"CC0"`, `"CC BY-SA 4.0"`, `"Public Domain"`) |
| `source` | string or null | no | Source attribution (e.g., `"The Metropolitan Museum of Art"`) |

---

# 4. Thumbnail Specs

**Why embed thumbnails?**
- Instant UI rendering (no network delay)
- Offline browsing
- Graceful API-failure fallback
- Consistent visual quality across collections

**Recommended format: WebP**
- 25-35% smaller than JPEG at equivalent quality
- Dimensions: 300-400px long edge
- Quality: 70-80%
- Target file size: 15-40 KB per thumbnail
- Encoding: Base64 (with or without `data:image/webp;base64,` prefix)

---

# 5. Complete Example

## 5.1 Minimal Collection File

```json
{
  "collectionId": "impressionism",
  "collectionName": "Impressionism",
  "description": "French Impressionist masterworks",
  "version": "1.0.0",
  "lastUpdated": "2025-06-15T00:00:00Z",
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
          "embeddedThumbnail": "BASE64_WEBP_THUMBNAIL",
          "embeddedThumbnailFormat": "image/webp",
          "license": "CC0",
          "source": "The Metropolitan Museum of Art"
        }
      ]
    }
  ]
}
```

## 5.2 Artwork with Pre-Resolved URLs

If you already have image URLs, provide them directly. The app will use them without making API calls:

```json
{
  "id": "met-436535",
  "museumId": "met",
  "sourceObjectId": "436535",
  "title": "The Harvesters",
  "artist": "Pieter Bruegel the Elder",
  "date": "1565",
  "medium": "Oil on wood",
  "classification": "Painting",
  "subjects": ["Landscape"],
  "imageURL": "https://images.metmuseum.org/CRDImages/ep/original/DP-42549-001.jpg",
  "thumbnailURL": "https://images.metmuseum.org/CRDImages/ep/web-large/DP-42549-001.jpg",
  "embeddedThumbnail": "data:image/webp;base64,UklGRiIAAABXRUJQ...",
  "embeddedThumbnailFormat": "image/webp",
  "license": "CC0",
  "source": "The Metropolitan Museum of Art"
}
```

## 5.3 Wikimedia Commons Artwork

For Wikimedia Commons, the `sourceObjectId` is the filename (without the `File:` prefix):

```json
{
  "id": "wikimedia-mona-lisa",
  "museumId": "wikimedia",
  "sourceObjectId": "Mona_Lisa,_by_Leonardo_da_Vinci,_from_C2RMF_retouched.jpg",
  "title": "Mona Lisa",
  "artist": "Leonardo da Vinci",
  "date": "ca. 1503-1519",
  "medium": "Oil on poplar panel",
  "classification": "Painting",
  "subjects": ["Portrait"],
  "imageURL": null,
  "thumbnailURL": null,
  "embeddedThumbnail": null,
  "embeddedThumbnailFormat": null,
  "license": "Public Domain",
  "source": "Wikimedia Commons"
}
```

---

# 6. File Naming

Collection files must be named `{collectionId}.paradata.json`:
- `masters.paradata.json`
- `impressionism.paradata.json`
- `landscapes.paradata.json`

---

# 7. What's NOT in the Collection JSON

These fields belong in the **Museum API Manifest** (`museum_apis.json`), not in collection files:

- `apiEndpoint`
- `apiType`
- `imageStrategy`
- `imageField`
- `thumbnailField`
- `iiifTemplate`
- `requiresApiKey`

The manifest handles all API logic and can be updated independently of collection data.

---

# 8. Supported Museum Sources

| `museumId` | Museum | `sourceObjectId` format | Resolution | License |
|-----------|--------|------------------------|------------|---------|
| `met` | The Metropolitan Museum of Art | Numeric object ID (e.g., `"436535"`) | API call returns image URLs | CC0 |
| `aic` | Art Institute of Chicago | Numeric artwork ID (e.g., `"27992"`) | API call returns IIIF image ID, URL constructed from template | CC0 |
| `nga` | National Gallery of Art | UUID (e.g., `"12345-abcd-..."`) | URL constructed directly from ID, no API call | CC0 |
| `wikimedia` | Wikimedia Commons | Filename without `File:` prefix (e.g., `"Mona_Lisa,_by_Leonardo_da_Vinci,_from_C2RMF_retouched.jpg"`) | MediaWiki API query | Various (CC BY-SA, CC0, PD) |

**Finding sourceObjectId values:**
- **Met:** The number in the URL, e.g., `metmuseum.org/art/collection/search/436535` -> `"436535"`
- **AIC:** The number in the URL, e.g., `artic.edu/artworks/27992` -> `"27992"`
- **NGA:** The UUID from the NGA API
- **Wikimedia:** The filename from the Commons page URL, e.g., `commons.wikimedia.org/wiki/File:Starry_Night.jpg` -> `"Starry_Night.jpg"`

---

# 9. Quick Reference: Artwork Object (All Fields)

```json
{
  "id": "string",                          // required - unique ID
  "museumId": "string",                    // required - see Section 8
  "sourceObjectId": "string",             // required for image resolution
  "title": "string",                       // required
  "artist": "string",                      // required
  "date": "string",                        // required
  "medium": "string",                      // required
  "classification": "string",             // required
  "subjects": ["string"],                  // required - at least one tag
  "imageURL": "string or null",           // optional - pre-resolved image URL
  "thumbnailURL": "string or null",       // optional - pre-resolved thumbnail URL
  "embeddedThumbnail": "string or null",  // optional - base64 thumbnail data
  "embeddedThumbnailFormat": "string or null",  // optional - e.g., "image/webp"
  "license": "string or null",            // optional - e.g., "CC0"
  "source": "string or null"              // optional - e.g., "The Metropolitan Museum of Art"
}
```
