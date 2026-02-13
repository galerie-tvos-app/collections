# Theme-Based Paradata JSON Specification with Nested Themes (v1.02) 
(API‑First, Museum‑Agnostic, With Embedded Thumbnails)

This specification defines the structure for theme collections, sub‑themes, and artworks.  
It is designed for a multi‑museum, API‑resolved image system with stable embedded thumbnails.

A *theme* may represent:

- A broad category (e.g., “Masters”, “Impressionism”, “Landscapes”)
- A sub-theme (e.g., “Van Gogh”, “Renoir”, “Dutch Golden Age”)
- A curated playlist
- A museum (optional)
- A user-created collection

Themes may include artworks from **any source**, including multiple museums or user-owned images.

---

# 1. Collection Object

```json
{
  "collectionId": "string",
  "collectionName": "string",
  "description": "string",
  "version": "string",
  "lastUpdated": "ISO-8601 timestamp",
  "subThemes": [ ... ]
}
```

---

# 2. Sub‑Theme Object

```json
{
  "subThemeId": "string",
  "subThemeName": "string",
  "description": "string",
  "items": [ ... ]
}
```

---

# 3. Artwork Object  
### *(Museum‑agnostic, API‑first, with embedded fallback thumbnails)*

Each artwork entry includes:

## A. Museum Identification  
These fields allow your backend to resolve images using the **Museum API Manifest**.

```json
{
  "id": "string",                 // internal unique ID (e.g., "met-436535")
  "museumId": "string",           // e.g., "met", "aic", "rijks", "nga"
  "sourceObjectId": "string",     // museum's object ID
```

No museum API logic appears here — that lives in the manifest.

---

## B. Artwork Metadata

```json
  "title": "string",
  "artist": "string",
  "date": "string",
  "medium": "string",
  "classification": "string",
  "subjects": ["string"],
```

---

## C. Runtime‑Resolved Fields  (Optional)
These may be **null** in the JSON file and values should not be trusted due to API changes.

```json
  "imageURL": null,               // resolved via museum API
  "thumbnailURL": null,           // resolved via museum API
```

---

## D. Embedded Stable Thumbnail (Required)  
This ensures UI stability even if:

- the museum API is down  
- the user is offline  
- rate limits are hit  
- the museum changes its image URLs  

```json
  "embeddedThumbnail": "string (base64-encoded WebP)",
  "embeddedThumbnailFormat": "image/webp"
}
```

---

# 4. Thumbnail Requirements

## 4.1 Why embed thumbnails?
- Instant UI rendering  
- Offline browsing  
- Graceful API‑failure fallback  
- No dependency on museum servers  
- Consistent visual quality  

## 4.2 Recommended Format  
### **Use WebP thumbnails**

**Why WebP?**
- 25–35% smaller than JPEG  
- Better quality at low sizes  
- Supported everywhere  
- Ideal for grid views  

## 4.3 Recommended Specs
- **Format:** `image/webp`  
- **Dimensions:** 300–400px long edge  
- **Quality:** 70–80%  
- **Target size:** 15–40 KB  
- **Encoding:** Base64  

---

# 5. Example Artwork Entry (Final Form)

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

  "imageURL": null,
  "thumbnailURL": null,

  "embeddedThumbnail": "data:image/webp;base64,UklGRiIAAABXRUJQVlA4IC4AAADwAQCdASoCAAIALmkA...",
  "embeddedThumbnailFormat": "image/webp"
}
```

---

# 6. What’s NOT in the Collection JSON  
These fields **do not belong** in the theme files:

- `apiEndpoint`  
- `apiType`  
- `imageStrategy`  
- `imageField`  
- `thumbnailField`  
- `iiifTemplate`  
- `requiresApiKey`  

All of these live in the **Museum API Manifest**, which is maintained separately and can evolve without touching your theme data.

---

# 7. Why This Architecture Works

- **Museum‑agnostic theme files**  
- **Single source of truth for API logic**  
- **Stable UI even during API failures**  
- **Efficient bandwidth usage**  
- **Future‑proof and scalable**  
- **Easy to add new museums**  
- **Easy to update API rules without touching artwork data**



