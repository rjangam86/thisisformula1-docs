# 08 — Image Handling & Media Strategy  
_ThisIsFormulaOne Platform — Image Workflow & CDN Optimization_

This document defines how article images are stored, delivered, optimized, cached, and referenced across the frontend and backend services.

---

## 1. Media Storage Strategy

All article images are stored in:

```
s3://tif1-media/images/articles/{article_id}.jpg
```

Folder structure:

- `/images/articles/` — Main article images  
- `/images/thumbs/` — Auto-resized thumbnails (Phase 2)  
- `/images/uploads/` — Raw uploads from admin tools  

Naming convention:

- `{article_id}.jpg` — Primary image  
- `{article_id}-thumb.jpg` — Thumbnail  
- `{article_id}-wide.jpg` — Optional widescreen header  

---

## 2. How the Website Loads Images

Frontend receives image URLs directly from the API:

```json
{
  "image_url": "https://cdn.thisisformulaone.net/images/articles/1763550009.jpg"
}
```

The API **never returns S3 URLs** — only CloudFront URLs.

---

## 3. CloudFront CDN Configuration

CDN Domain:

```
https://cdn.thisisformulaone.net
```

Settings:

| Setting | Value |
|--------|--------|
| Origin | S3 Bucket |
| Cache TTL | 24 hours default |
| Compression | GZIP + Brotli |
| Viewer Protocol | HTTPS only |
| CORS | Allowed for all website origins |

Image caching rules:

- Query-string ignored  
- Content hashed globally  
- All images immutable  

---

## 4. Lambda Integration (Read Only)

The read API (GET /articles) simply returns:

- `image_url`  
- `image_source`  
- `image_alt` (Phase 2)  

No resizing or processing occurs inside Lambda.

---

## 5. Thumbnails (Phase 2)

We will generate thumbnails automatically using:

- AWS Lambda (Sharp)  
- or AWS S3 Object Lambda (recommended)

Thumbnail paths:

```
/images/thumbs/{article_id}.jpg
```

Frontend uses thumbnails ONLY in:

- More News Grid  
- Sidebar Latest Posts  
- Category Pages  
- Tag Pages  

---

## 6. Image Optimization Rules

Guidelines:

- Images should be **≤ 200 KB** ideally  
- Format: **JPEG** for photos, **WEBP** optional in Phase 3  
- Minimum resolution: **1200px width**  
- No PNG unless transparency needed  

Backend stores:

| field | type | description |
|-------|-------|-------------|
| `image_url` | String | CDN URL |
| `image_source` | String | “PlanetF1”, “F1”, “Ferrari Media” |
| `image_alt` | String | Auto-filled or manual |

---

## 7. Fallback Image

If an article has **no image**, use:

```
https://cdn.thisisformulaone.net/images/placeholder/default.jpg
```

Frontend logic:

- If `image_url` missing → display placeholder  
- Never break layout  

---

## 8. Multiple Images (Phase 3)

Future support for galleries:

- `/v1/articles/{id}/media`
- Ordered list of images  
- Lightbox gallery on frontend  

Not required for launch.

---

## 9. Responsive Image Guidelines

Use `<img>` tags with:

```html
<img src="IMAGE_URL"
     loading="lazy"
     class="img-fluid"
     alt="Race weekend news">
```

Lazy loading improves mobile performance drastically.

---

## 10. Security Rules

- All images must be public-read via CloudFront  
- S3 bucket must NOT be publicly accessible  
- No direct S3 URLs in API  
- Website must only use CDN domain  

---

## 11. Required Frontend Changes

Add image loading logic in:

| Page | Script |
|------|--------|
| Homepage | `render-home.js` |
| Article Detail | `render-article.js` |
| Category Page | `render-category.js` |
| Tag Page | `render-tag.js` |

Each renderer will:

1. Insert article thumbnail  
2. Insert full image on detail page  
3. Add fallback image if broken

---

## 12. Related Documents

- `01-overview.md`  
- `02-dynamodb-schema.md`  
- `03-api-gateway-design.md`  
- `04-lambda-read-api.md`  
- `05-frontend-integration.md`  
- `06-auth-and-users.md`  
- `07-site-structure.md`

---