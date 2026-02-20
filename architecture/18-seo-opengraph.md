# 11 — Search, SEO & OpenGraph Master Guide  
_ThisIsFormulaOne Platform — Search, Discoverability, Metadata, OpenGraph, Routing & Sitemaps_

This document is the **single authoritative specification** for:

- Search system (Phase 1 + Phase 2)
- Tag/type/keyword endpoints
- SEO metadata requirements
- OpenGraph + Twitter Card requirements
- URL routing rules
- Sitemaps & robots.txt
- Performance caching for SEO pages

It replaces and merges all previous versions of:
- **11-search-and-seo.md**
- **18-seo-opengraph.md**

---

# 1. Overview

Search, SEO, and sharing metadata are critical pillars:

- **Search** helps users find articles and improves engagement.
- **SEO** brings organic traffic from Google.
- **OpenGraph** ensures your content looks premium when shared on Facebook/Threads/Twitter/WhatsApp.

This guide ensures your website is fully discoverable and optimized.

---

# 2. Search Architecture (Phase 1)

Phase 1 is intentionally simple and cost-efficient:

| Feature | Status | Notes |
|---------|--------|-------|
| Keyword search | Enabled | DynamoDB Scan + FilterExpression |
| Tag search | Enabled | `/v1/articles/by-tag/{tag}` |
| Category/type search | Enabled | `/v1/articles/by-type/{type}` |
| Full-text search | Phase 2 | Requires dedicated index |
| Auto-suggest | Phase 2 | Optional |

Supported in Phase 1:

- Driver tags (`/lewishamilton`)
- Team tags (`/ferrari`)
- Topic tags (`contracts`, `engine`, `strategy`)
- Generic text queries (`/search?q=verstappen`)

CloudFront caching ensures performance despite DynamoDB scans.

---

# 3. API Endpoints for Search

## 3.1 Tag Search  
```
GET /v1/articles/by-tag/{tag}
```
Returns all articles that include a tag.

- Sorted by `published_at` DESC  
- Paginated (`page`, `per_page`)  
- DynamoDB Scan + Filter (fast because cached)

---

## 3.2 Type Search (Category Pages)
```
GET /v1/articles/by-type/{article_type}
```

Supported article types:

- `breaking`
- `latest`
- `tech`
- `analysis`
- `race_preview`
- `race_debrief`
- `featured`

Uses **GSI_ByType**:  
PK = `article_type`, SK = `published_at` DESC.

---

## 3.3 Keyword Search (Basic Mode)
```
GET /v1/search?q=verstappen
```

Logic:

1. Normalize query  
2. Search:
   - title
   - excerpt
   - tags
3. Limit results to **20**

Performance technique:
- CloudFront caching (30–60 sec TTL)

---

# 4. Full-Text Search (Phase 2)

To enable real search, create:

## DynamoDB FTS Table
```
F1ArticlesIndex
  PK: keyword
  SK: published_at
  Attributes:
     article_id
     score
```

Allows:

- Ranked relevance search  
- Driver/team/topic indexing  
- Auto-suggest  
- Fast boolean queries (keyword + tag combination)

Keyword extraction strategy:

- Tokenize title  
- Tokenize tags  
- Tokenize driver/team names  
- Tokenize content (optional)

---

# 5. SEO Metadata Architecture

Every article page must dynamically inject:

```html
<title>
<meta name="description">
<link rel="canonical">
<meta name="keywords">
<meta name="author">
<meta name="robots">
```

### Example:
```html
<title>Leclerc: “Ferrari está no caminho certo” – This Is Formula One</title>
<meta name="description" content="Charles Leclerc fala sobre confiança na Ferrari e planos para 2026.">
<link rel="canonical" href="https://thisisformulaone.net/articles/leclerc-ferrari-2026">
<meta name="keywords" content="Leclerc, Ferrari, F1, Formula 1 news">
<meta name="author" content="This Is Formula One Editorial">
<meta name="robots" content="index, follow">
```

Metadata must be derived from:

- Title  
- Excerpt  
- Tags  
- Slug  
- Article type  

---

# 6. OpenGraph + Twitter Cards

Every article should embed:

```html
<meta property="og:title" content="Leclerc signs new Ferrari contract">
<meta property="og:description" content="Charles Leclerc extends his deal with Ferrari.">
<meta property="og:image" content="https://cdn.thisisformulaone.net/images/articles/123.jpg">
<meta property="og:url" content="https://thisisformulaone.net/articles/leclerc-signs-extension">
<meta property="og:site_name" content="This Is Formula One">
<meta property="og:type" content="article">

<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:title" content="Leclerc signs new Ferrari contract">
<meta name="twitter:description" content="Charles Leclerc extends his contract with Ferrari.">
<meta name="twitter:image" content="https://cdn.thisisformulaone.net/images/articles/123.jpg">
```

Platforms enhanced:

- Facebook  
- Instagram Threads  
- WhatsApp previews  
- Twitter/X  
- LinkedIn  

Benefits:

- Higher CTR  
- Better shareability  
- More traffic on breaking news  

---

# 7. URL Structure & Routing

Canonical URL:

```
https://thisisformulaone.net/articles/{slug}
```

Examples:

```
/articles/sainz-merc-2026
/articles/verstappen-pole-suzuka
/articles/hamilton-brazil-masterclass
```

Guidelines:

- Always lowercase  
- Hyphens only  
- No dates in URL  
- Slug derived from title  
- No article ID in URL (but kept internally)  

---

# 8. Sitemap Strategy

Expose:

```
/sitemap.xml
/sitemap-articles.xml
/sitemap-tags.xml
/sitemap-categories.xml
```

Regenerated daily via Lambda:

- Up to 50,000 URLs per file  
- Ensures Googlebot finds new pages instantly  

---

# 9. Robots.txt

```
User-agent: *
Allow: /

Sitemap: https://thisisformulaone.net/sitemap.xml
```

---

# 10. Frontend SEO Requirements

Frontend must:

- Insert metadata **before** rendering  
- Preload primary article image  
- Lazy-load everything else  
- Use `<h1>` only for titles  
- Use `<h2>` for sections  
- Avoid layout shifts  
- Prioritize LCP (Largest Contentful Paint)

Core metrics:

- Lighthouse SEO ≥ 95  
- All Web Vitals = green  

---

# 11. Performance & Caching (SEO Focus)

| Layer | TTL | Notes |
|-------|------|-------|
| **HTML** | 0 | Always dynamic |
| **JSON API** | 30–60 sec | Prevents DDB spikes |
| **Images** | 1 day | Cached via CloudFront |
| **Tag pages** | 5 min | Updated often |
| **Category pages** | 5 min | Same logic |

---

# 12. Tag Standardization Rules

Tags fuel both:

- Search accuracy  
- Internal routing  
- SEO keyword mapping

Rules:

- Lowercase only  
- Hyphenate spaces  
- Driver tags = full name (`lewis-hamilton`)  
- Team tags = canonical name (`mercedes`, `ferrari`)  
- Topic tags = simple (`contracts`, `strategy`, `upgrade`)  

These tags appear in:

- API responses  
- Slugs  
- Frontend filters  
- Sitemaps  

---

# 13. Data Requirements From DynamoDB

Every article must include:

| Field | Used For |
|-------|----------|
| title | `<title>` + OpenGraph |
| excerpt | Meta description |
| image_url | OpenGraph/Twitter |
| slug | Canonical URL |
| tags | Tag pages + search |
| article_type | Category pages |
| published_at | Sorting |
| content | Article body |

---

# 14. Related System Documents

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