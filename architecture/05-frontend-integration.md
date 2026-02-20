# 05 — Frontend Integration  
_ThisIsFormulaOne Platform — Connecting API to UI_

This document defines how the **existing HTML/CSS template** integrates with the new REST API and how the frontend will dynamically load articles, previews, categories, and tags using JavaScript (jQuery or Vanilla JS).

---

# 1. Overview

The frontend:
- Uses the existing ThemeForest MinBeriMag layout (sports version).
- Loads article content dynamically from:  
  `https://api.thisisformulaone.net/v1`
- Renders:
  - Homepage sections (Breaking, Latest, Tech, Featured)
  - Category pages
  - Tag pages
  - Single article pages
  - Pagination
- Runs 100% client-side (JavaScript), no server rendering.

---

# 2. API Endpoints Used by Frontend

| Purpose | Endpoint |
|---------|----------|
| Homepage feed | `GET /v1/articles` |
| Category pages | `GET /v1/articles/by-type/{type}` |
| Tag pages | `GET /v1/articles/by-tag/{tag}` |
| Full article page | `GET /v1/articles/{article_id}` |
| Search (Phase 2) | `GET /v1/search?q=` |

---

# 3. Frontend File Mapping

Your uploaded template (`index.html`) contains:

| UI Section | API Source |
|-----------|-------------|
| Breaking News Block | `/v1/articles/by-type/breaking` |
| Latest News Block | `/v1/articles?page=1` |
| Featured Block | `/v1/articles?page=1` (featured = first item) |
| “More News / Tech” Block | `/v1/articles/by-type/tech` |
| Sidebar → Tags | Click → `/v1/articles/by-tag/{tag}` |
| Single Article Page | `/v1/articles/{id}` |

Other sections (Drivers, Teams, Results) will come later.

---

# 4. Homepage Rendering Workflow

## 4.1 Steps
1. Browser loads `index.html`
2. JavaScript runs `loadHomepage()`
3. API request:  
   `GET https://api.thisisformulaone.net/v1/articles?page=1`
4. Extract:
   - `featured = data.data.featured`
   - `previews = data.data.previews`
5. Render:
   - First preview as main “Breaking/featured headline”
   - Other 5 previews into cards below
6. Run breaking + tech sections.

---

# 5. Pagination Handling

Pagination UI (Page 1, 2, 3…) uses:

```
GET /v1/articles?page=2
```

Rules:
- Page 1 → featured + 5 previews  
- Page > 1 → only previews (no featured)

Displayed in your “More News” section at bottom.

---

# 6. Category Pages

Links such as:
```
breaking.html
tech.html
analysis.html
results.html
```

Each page will run:

```
GET /v1/articles/by-type/{type}?page=X
```

And render the same layout:
- Featured (page 1 only)
- Previews
- Pagination

---

# 7. Tag Pages

Sidebar tag → example:

```
/v1/articles/by-tag/lewishamilton
```

Render same layout as category:
- Featured (first result)
- Previews
- Pagination

---

# 8. Single Article Page Rendering

The template page for a full article will:

1. Extract the article ID from URL:  
   Example: `article.html?id=12345`
2. API call:
   ```
   GET /v1/articles/12345
   ```
3. Render:
   - Title
   - Published date
   - Article content (HTML)
   - Image
   - Image source
   - Article source
   - Tags (clickable)
4. Under article → show related posts:
   - Based on same type OR same tags.

---

# 9. Frontend Folder Structure

Recommended structure:

```
/assets/js/api.js        → all API functions
/assets/js/render.js     → UI render functions
/assets/js/home.js       → homepage logic
/assets/js/article.js    → single article logic
/assets/js/category.js   → category pages
/assets/js/tag.js        → tag pages

index.html
article.html
breaking.html
tech.html
analysis.html
results.html
tags.html
```

---

# 10. Shared JavaScript Utilities

### `apiGet(url)`
Generic fetch helper:

```javascript
async function apiGet(url) {
  const res = await fetch(url);
  return await res.json();
}
```

### Dynamic Render Functions
- `renderFeatured(article)`
- `renderPreviewList(previews)`
- `renderPagination(meta)`
- `renderTagsSidebar(tags)`

These functions plug directly into your HTML structure.

---

# 11. What You Will Provide

You (Rahul) will supply:
- `index.html` (done)
- The additional files referenced:
  - `/assets/js/*.js`
  - `/partials/header.html`
  - `/partials/sidebar.html`
  - `/partials/footer.html`
- The installed dependencies (if any).

After providing them, we integrate the code directly into your template.

---

# 12. Backend Requirements for Frontend

Frontend needs:
- Stable API URL
- CORS correctly configured
- 6 items per page
- Each article object must include:
  - `article_id`
  - `title`
  - `excerpt`
  - `article_type`
  - `tags`
  - `published_at`
  - `image_url`
  - `image_source`
  - `article_source`
  - `slug`
  - `content` (for full article page)
- Thumbnail-sized image (same URL is okay)

---

# 13. Related Documents

- `01-overview.md`
- `02-dynamodb-schema.md`
- `03-api-gateway-design.md`
- `04-lambda-read-api.md`
- `07-site-structure.md`

---