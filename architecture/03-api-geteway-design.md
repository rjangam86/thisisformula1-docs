# 03 — API Gateway Design  
_ThisIsFormulaOne Platform — REST API Layer_

This document defines the REST API architecture, endpoints, pagination logic, query filters, URL conventions, and backend integration used by the ThisIsFormulaOne website.

---

## 1. API Gateway Overview

| Property | Value |
|---------|--------|
| **API Type** | REST API (not HTTP API) |
| **Base URL** | `https://api.thisisformulaone.net/v1` |
| **Auth** | None (public read-only) |
| **CORS** | Allowed: `https://thisisformulaone.net`, `http://localhost:3000` |
| **Stage** | `prod` |
| **Integration** | Lambda Proxy Integration |
| **Timeout** | 29 seconds |
| **Caching** | CloudFront + API Gateway optional caching (future) |

---

## 2. Endpoint Summary

| Method | Route | Description |
|--------|--------|-------------|
| **GET** | `/v1/articles` | Homepage feed with pagination |
| **GET** | `/v1/articles/{article_id}` | Full article content |
| **GET** | `/v1/articles/by-type/{type}` | Category pages (breaking, tech, results…) |
| **GET** | `/v1/articles/by-tag/{tag}` | Tag pages (Ferrari, Hamilton…) |
| **GET** | `/v1/search` | Multi-field search (Phase 2) |

All endpoints return **JSON envelopes** with metadata.

---

## 3. Standard Response Envelope

```json
{
  "data": {},
  "meta": {
    "page": 1,
    "per_page": 6,
    "total_pages": 10,
    "total_items": 59
  }
}
```

---

## 4. Pagination Logic

### Items Per Page
- Homepage feed uses **6 items per page**
    - 1 featured article (full)
    - 5 previews (compact)

### Cursor Behavior
- `page` is 1-indexed.
- `per_page` can be overridden but defaults to 6.

### DynamoDB Query Model
- Sorted by `published_at` (descending).
- Pagination is done with `Limit` + `ExclusiveStartKey`.

### Meta Block Always Returned
- `page`
- `per_page`
- `total_pages`
- `total_items`

---

## 5. Query Filters

### Query Parameters (Homepage)
| Param | Purpose |
|-------|----------|
| `page` | Page number |
| `per_page` | Items per page |
| `type` | Filter by `article_type` |
| `tag` | Filter by tag list |

### Filter Logic
- `type` queries use **GSI_ByType**
- `tag` queries use:
  - Phase 1 → **FilterExpression**
  - Phase 2 → dedicated `TagIndex` inverted index (planned)

---

## 6. Homepage Feed (`GET /v1/articles`)

Returns:
- 1 featured article (full details)
- 5 preview articles

### Sample Response
```json
{
  "data": {
    "featured": {
      "article_id": "123",
      "title": "Leclerc signs extension",
      "excerpt": "Charles Leclerc has committed his future...",
      "image_url": "https://...",
      "article_type": "breaking",
      "published_at": 1763512301,
      "slug": "leclerc-signs-extension"
    },
    "previews": [
      {
        "article_id": "124",
        "title": "Verstappen on pole",
        "excerpt": "Max Verstappen secured pole position...",
        "image_url": "https://...",
        "article_type": "results",
        "published_at": 1763512200,
        "slug": "verstappen-pole-suzuka"
      }
    ]
  },
  "meta": {
    "page": 1,
    "per_page": 6,
    "total_pages": 15,
    "total_items": 88
  }
}
```

---

## 7. Category Pages (`GET /v1/articles/by-type/{type}`)

Examples of supported types:
- `breaking`
- `tech`
- `results`
- `analysis`
- `interview`
- `feature`

### Behavior
- Returns compact previews only (no full article).
- Sorted by newest first.
- Uses GSI: `GSI_ByType`.

---

## 8. Tag Pages (`GET /v1/articles/by-tag/{tag}`)

Examples:
- `/v1/articles/by-tag/Ferrari`
- `/v1/articles/by-tag/Hamilton`
- `/v1/articles/by-tag/Leclerc`

Tag matching:
- Phase 1 → uses FilterExpression on DynamoDB results.
- Phase 2 → full inverted index for perfect tag performance.

---

## 9. Single Article (`GET /v1/articles/{article_id}`)

Returns full article data:

```json
{
  "data": {
    "article_id": "123",
    "title": "Leclerc signs extension",
    "content": "<p>Full HTML content...</p>",
    "image_url": "https://...",
    "image_source": "Ferrari Media",
    "article_source": "PlanetF1",
    "published_at": 1763512301,
    "article_type": "breaking",
    "tags": ["Ferrari", "Contracts"],
    "language": "en",
    "author": "Staff"
  }
}
```

---

## 10. Error Handling Rules

### Standard Error Envelope
```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "Article not found"
  }
}
```

### Error Codes
- `NOT_FOUND`
- `INVALID_QUERY`
- `INTERNAL_ERROR`

---

## 11. Performance & Caching

### CloudFront
- 60–120 second TTL for `/v1/articles`
- Longer TTL (300–600s) for article pages

### API Gateway
- Optional 60s cache for category & tag pages

### DynamoDB
- Strict use of GSIs
- Minimal FilterExpressions in Phase 1

---

## 12. Related Documents

- `02-dynamodb-schema.md`
- `04-lambda-read-api.md`
- `05-frontend-integration.md`
- `06-seo-opengraph.md`
- `07-site-structure.md`

---