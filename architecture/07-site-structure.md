# 07 — Website Structure & Page Layout  
_ThisIsFormulaOne Platform — Frontend Architecture (Based on Uploaded Template)_

This document defines the **page structure, navigation, section mapping**, and how each part of the HTML template will eventually connect to API data.  
It is based fully on your uploaded `index.html` template.

---

# 1. Global Page Structure

Your template uses a **modular layout** with:

- **Header** (top bar)
- **Main Navigation Menu**
- **Breaking News Strip** (optional)
- **Homepage Sections** (multiple)
- **Sidebar Widgets**
- **Footer**
- **Article Detail Page Template** (to be added)

The website renders:

- One **featured article**
- Several **preview articles**
- A “More News” grid with pagination
- A **sidebar** with:
  - Tags
  - Latest News Widget
  - Social Links
  - Featured Posts

We will integrate ALL of these with real DynamoDB data.

---

# 2. Pages to Build

You will end up with **6 pages**:

### **1. Homepage**  
`/index.html`  
Sections:
- Featured Article (1)
- Highlights Section (3–4 articles)
- Latest News Section (5–8 articles)
- Tech Features (3 articles)
- More News Grid (Paginated)

---

### **2. Article Detail Page** *(New file)*  
`/article.html?id={article_id}`  
Shows:
- Full Article
- Title, date, image, source
- Breadcrumb: Home → Category → Article
- Sidebar widgets (same as homepage)
- Prev/Next article navigation
- Tags (clickable)

---

### **3. Category Page**  
`/category.html?type={type}`  
Examples:
- breaking
- tech
- results
- analysis
- interviews  
Layout:
- Article list based on type
- Pagination bottom
- Sidebar widgets

---

### **4. Tag Page**  
`/tag.html?tag={tag_name}`  
Examples:
- Hamilton
- Leclerc
- Ferrari  
Layout: same as category page.

---

### **5. Search Page**  
`/search.html?q={query}`  
(Search API is Phase 2)

---

### **6. Contact Page**  
`/contact.html`  
Static for now.

---

# 3. Navigation Menu Mapping

Your current menu:

- **Latest News**
- **Drivers**
- **Teams**
- **Results**
- **Tech Analysis**
- **Shop**
- **Contact**

We will map them:

| Menu | Destination |
|------|-------------|
| Latest News | `/category.html?type=latest` |
| Drivers | (Phase 3 – driver stats) |
| Teams | (Phase 3 – team stats) |
| Results | `/category.html?type=results` |
| Tech Analysis | `/category.html?type=tech` |
| Shop | External Shopify URL |
| Contact | `/contact.html` |

Nothing will break even if not implemented yet.

---

# 4. Homepage Data Sections

Homepage will fetch:

```
GET /v1/articles?page=1&per_page=20
```

Then we **partition** the items:

### **1 Featured Article**
- Most recent breaking OR general article

### **3–4 Spotlight Articles**  
Pull from the next few results.

### **Latest News List (5–8 items)**
Sorted by published_at.

### **Tech Corner (3 items)**
From article_type = tech

### **More News Grid**  
Paginated via:
```
GET /v1/articles?page=1&per_page=6
```

Pagination displayed under the grid.

---

# 5. Article Detail Page Structure

When user opens:
```
/article.html?id=1763550009
```

Frontend loads:
```
GET /v1/articles/{id}
```

Renders:

- Title
- Published date
- Main image
- Article source link
- Full HTML content
- Tags (clickable → tag page)
- Sidebar widgets

---

# 6. Sidebar Architecture

Works on ALL pages (homepage, category, article).

### Widgets:

#### A. Social Links  
Static.

#### B. Latest Posts  
API:
```
GET /v1/articles?per_page=5
```

#### C. Tags  
Static list (initially).  
Phase 2 → convert to dynamic tags from DB.

#### D. Featured Post Widget  
Use:
```
GET /v1/articles/by-type/highlight?per_page=1
```

---

# 7. Footer

Footer remains static:
- About Section
- Social Icons
- Category Links (Breaking, Tech, etc.)
- Newsletter Subscription (optional future)

---

# 8. URL Routing Strategy

You will NOT use SPA or React.  
HTML files + jQuery + API calls.

Typical navigation:

```
index.html
article.html?id=xxxx
category.html?type=tech&page=2
tag.html?tag=ferrari
search.html?q=verstappen
```

Each page loads the relevant JSON and fills containers.

---

# 9. Required New HTML Files

You need to create:

- `article.html`  
- `category.html`  
- `tag.html`  
- `search.html`  

Each based on your main template:
- Copy layout
- Replace content area only

---

# 10. Required JS Files (To Create)

```
/js/api.js             // API wrapper
/js/render-home.js     // Homepage rendering
/js/render-article.js  // Article page
/js/render-category.js // Category page
/js/render-tag.js      // Tag page
/js/render-search.js   // Search
```

Inside **api.js**, define helpers:

```js
const API_BASE = "https://api.thisisformulaone.net/v1";

async function fetchJSON(path) {
    const res = await fetch(`${API_BASE}${path}`);
    return await res.json();
}
```

---

# 11. Optional Later Enhancements

- Polls Widget
- Commenting system
- User login
- Infinite scroll for mobile
- Dark mode
- Real-time race updates widget

This will integrate later without reworking layout.

---

# 12. Related Documents

- `01-overview.md`
- `02-dynamodb-schema.md`
- `03-api-gateway-design.md`
- `04-lambda-read-api.md`
- `05-frontend-integration.md`
- `06-auth-and-users.md`

---