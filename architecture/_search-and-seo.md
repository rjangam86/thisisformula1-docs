# 11 — Search & SEO Strategy  
_ThisIsFormulaOne Platform — Search, Discoverability & Metadata Optimization_

This document defines the **search system**, **SEO metadata architecture**, **OpenGraph/Twitter Cards**, and **URL strategy** for the website.  
It covers Phase-1 (basic search + SEO) and the future Phase-2 (full search engine).

---

# 1. Overview

Search & SEO are core pillars of the platform:

- **Search** helps readers find articles.
- **SEO** brings new visitors from Google.
- **OpenGraph** ensures posts look perfect on Facebook, Instagram Threads, X/Twitter, etc.

This plan ensures your website is fully ready for both.

---

# 2. Search Architecture (Phase 1)

Phase 1 is intentionally lightweight:

| Feature | Status | Notes |
|---------|--------|-------|
| Keyword search | Enabled | Uses DynamoDB + FilterExpression |
| Tag search | Enabled | `/v1/articles/by-tag/{tag}` |
| Category/type search | Enabled | `/v1/articles/by-type/{type}` |
| Full-text search | Phase 2 | Requires new index/table |
| Auto-suggest | Phase 2 | Optional |

Phase-1 search is designed to support:

- Driver tags (`/lewishamilton`)
- Team tags (`/ferrari`)
- Topic tags (`contracts`, `engine`, `updates`)
- Generic text search (`search?q=piastri`)

---

# 3. API Endpoints for Search

### 3.1 Tag Search
```
GET /v1/articles/by-tag/{tag}
```

Returns all articles containing this tag:

- Sorted by `published_at` DESC  
- Paginated (`page`, `per_page`)  
- Uses DynamoDB **Scan + Filter** (acceptable due to CloudFront caching)

---

### 3.2 Type Search (Category Pages)
```
GET /v1/articles/by-type/{article_type}
```

Possible `article_type` values:

- `breaking`
- `latest`
- `tech`
- `analysis`
- `race_preview`
- `race_debrief`
- `featured`

This uses `GSI_ByType` for fast queries.

---

### 3.3 Keyword Search (Basic Mode)
```
GET /v1/search?q=verstappen
```

Logic:

1. Normalize query (lowercase, trim)
2. Look into:
   - Title
   - Excerpt
   - Tags
3. Return first **20** matches

Performance mitigations:

- Cached by CloudFront for 30–60 sec
- Lambda returns a small footprint (title + tag match)

---

# 4. Full-Text Search (Phase 2)

To fully support advanced search:

### New DynamoDB FTS Table

```
F1ArticlesIndex
  PK: keyword
  SK: published_at
  Attributes:
     article_id
     score
```

This allows:

- Auto-suggest
- Precision matching
- Ranked relevance
- Faster tag + keyword filter combinations

Keywords are extracted via:

- Tokenizing title
- Tokenizing tags
- Tokenizing driver/team names
- Tokenizing content (optional)

This is a Phase-2 enhancement; current system does **not** need it.

---

# 5. SEO Metadata Architecture

Each article page must dynamically inject:

```
<title>
<meta name="description">
<link rel="canonical">
<meta name="keywords">
<meta name="author">
<meta name="robots">
```

Example:

```html
<title>Leclerc: "Ferrari está no caminho certo" – This Is Formula One</title>
<meta name="description" content="Charles Leclerc fala sobre temporada e confiança no projeto da Ferrari.">
<link rel="canonical" href="https://thisisformulaone.net/articles/leclerc-ferrari-2026">
<meta name="keywords" content="Leclerc, Ferrari, F1, F1 news, Formula 1 breaking news">
<meta name="author" content="This Is Formula One Editorial">
<meta name="robots" content="index, follow">
```

All metadata must be driven from:

- Published title
- Excerpt
- Tags
- Slug
- Article type

---

# 6. OpenGraph + Twitter Cards

Every article should include:

```html
<meta property="og:title" content="Leclerc signs new Ferrari contract">
<meta property="og:description" content="Charles Leclerc has extended his deal with Ferrari through 2029.">
<meta property="og:image" content="https://cdn.thisisformulaone.net/images/articles/123.jpg">
<meta property="og:url" content="https://thisisformulaone.net/articles/leclerc-signs-extension">
<meta property="og:site_name" content="This Is Formula One">
<meta property="og:type" content="article">

<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:title" content="Leclerc signs new Ferrari contract">
<meta name="twitter:description" content="Charles Leclerc extends his contract with Ferrari.">
<meta name="twitter:image" content="https://cdn.thisisformulaone.net/images/articles/123.jpg">
```

Benefits:

- Rich previews on Facebook & Threads
- Better CTR from WhatsApp shares
- Higher Twitter/X visibility

---

# 7. URL Structure & Routing

Canonical URL for articles:

```
https://thisisformulaone.net/articles/{slug}
```

Examples:

```
/articles/leclerc-renova-ferrari-2029
/articles/verstappen-pole-suuzka
/articles/perez-mclaren-2026
```

Guidelines:

- All lowercase
- Hyphens only
- No dates in the URL
- No IDs in the canonical link (but OK to use ID internally)

---

# 8. Sitemap Strategy

You will expose:

```
/sitemap.xml
/sitemap-articles.xml
/sitemap-tags.xml
/sitemap-categories.xml
```

Generated daily via Lambda or on-demand:

- Up to 50,000 URLs per sitemap
- Googlebot will crawl faster due to clean structure

---

# 9. Robots.txt

```
User-agent: *
Allow: /

Sitemap: https://thisisformulaone.net/sitemap.xml
```

---

# 10. Frontend SEO Requirements

The frontend must:

- Set metadata dynamically via JS **before page render**  
- Include preload of main image
- Lazy-load images below the fold (SEO friendly)
- Use `<h1>` only for article title
- Use `<h2>` for subsection headers

Critical metrics:

- Lighthouse SEO ≥ 95
- Core Web Vitals passing

---

# 11. Performance & Caching

| Layer | TTL | Notes |
|------|-----|-------|
| HTML | 0 | Always dynamic |
| JSON API | 30–60 sec | Prevents DynamoDB spikes |
| Images | 1 day | Safe & stable |
| Tags pages | 5 min | Medium volatility |
| Category pages | 5 min | Updated often |

---

# 12. Related Documents

- `01-overview.md`
- `02-dynamodb-schema.md`
- `03-api-gateway-design.md`
- `04-lambda-read-api.md`
- `05-frontend-integration.md`
- `06-auth-and-users.md`
- `07-site-structure.md`
- `08-image-handling.md`
- `09-caching-strategy.md`
- `10-deployment-guide.md`

---