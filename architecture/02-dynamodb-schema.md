# 02 — DynamoDB Schema & Indexes  
_ThisIsFormulaOne Platform — Data Storage Layer_

This document defines the complete DynamoDB schema for the Formula 1 news platform, including table structure, attribute definitions, index design, query patterns, and future extensibility for comments, users, and polls.

---

# 1. Core Table: `F1Articles`

All articles—breaking news, race previews, tech analysis, results, interviews—are stored in one unified table.  
This optimizes for:
- simple API design
- fast querying
- efficient paging
- easy multi-language support later
- low cost

---

# 2. Table Definition

| Attribute | Type | Required | Description |
|----------|------|----------|-------------|
| **article_id** | String (PK) | YES | Unique ID (e.g., `1763512301-lec-ext`) |
| **slug** | String | YES | SEO-friendly slug (`leclerc-signs-extension`) |
| **title** | String | YES | Article headline |
| **excerpt** | String | YES | The preview section / short summary |
| **content** | String | YES | Full HTML content (rendered on article page) |
| **image_url** | String | YES | Hero image URL (S3 + CloudFront) |
| **image_source** | String | YES | e.g., `Ferrari Media`, `F1`, `McLaren` |
| **article_source** | String | YES | e.g., PlanetF1, RacingNews365, Motorsport.com |
| **article_type** | String | YES | `breaking`, `tech`, `latest`, `analysis`, `race_preview`, `race_recap`, `results`, `interview` |
| **tags** | List<String> | OPTIONAL | e.g., `["hamilton", "mercedes"]` |
| **published_at** | Number | YES | Epoch timestamp |
| **status** | String | YES | Always `"PUBLISHED"` for public content |
| **language** | String | YES | e.g., `"en"`, `"pt-br"` |
| **author** | String | OPTIONAL | e.g., `"Staff"` |

---

# 3. Required Indexes (GSI Architecture)

The platform depends on three high-performance GSIs.

---

## 3.1 **GSI_AllArticles**  
For latest news sorted by publish time.

**Purpose:**  
- Homepage feed (`GET /v1/articles`)
- Pagination by most recent articles

**Definition:**

| Key | Value |
|------|--------|
| **PK** | `status` (value: `"PUBLISHED"`) |
| **SK** | `published_at` (Number, descending) |

**Query example:**

- `Query PK = "PUBLISHED"`  
- `ScanIndexForward = false` → newest first  
- `Limit = 6` for homepage  

---

## 3.2 **GSI_ByType**  
For category pages (Breaking News, Tech, Analysis).

**Purpose:**  
- `/v1/articles/by-type/{type}`  
- Pagination inside categories  
- Tech section, Breaking section, etc.

**Definition:**

| Key | Value |
|------|--------|
| **PK** | `article_type` |
| **SK** | `published_at` |

**Example Queries:**
- Breaking News: `article_type = "breaking"`
- Tech: `article_type = "tech"`

Sorting newest → oldest.

---

## 3.3 **GSI_TagsIndex**  
For hashtag/category searches.

**Purpose:**  
- `/v1/articles/by-tag/{tag}`
- Sidebar tag cloud clicks
- SEO traffic from tag pages

**Approach:**  
Because DynamoDB cannot index list attributes directly, we store a **projection row per tag**, i.e.:

### **Tag Index Row Format**
Stored inside **F1Articles** table:

| Attribute | Content |
|----------|----------|
| **PK** | `tag:<tag_name>` |
| **SK** | `published_at` |
| **article_id** | same article_id |
| **title** | article title |
| **excerpt** | preview |
| **slug** | slug |
| **image_url** | hero image |

These rows are extremely small and cheap.

**Example:**
A Hamilton article:

[“hamilton”, “mercedes”]

Creates two index rows:

PK = tag:hamilton
SK = 1763512301

PK = tag:mercedes
SK = 1763512301

This enables lightning-fast tag filtering.

---

# 4. Additional Tables (Future Features)

These will not be implemented now, but are pre-designed so the architecture stays scalable.

---

## 4.1 Table: `F1Users` (for comments in future)

| Attribute | Type |
|----------|------|
| user_id | String (PK) |
| email | String |
| password_hash | String |
| created_at | Number |

**Note:**  
Authentication can later be upgraded to Cognito.

---

## 4.2 Table: `F1Comments` (future)

| Attribute | Type |
|----------|------|
| comment_id | String (PK) |
| article_id | String |
| user_id | String |
| text | String |
| created_at | Number |
| status | String (`pending`, `approved`) |

Indexes later:

- GSI by `article_id`  
- GSI by `user_id`  

---

## 4.3 Table: `F1Polls` (deferred)

| Attribute | Type |
|----------|------|
| poll_id | String (PK) |
| question | String |
| options | List |
| votes | Map |

Not needed for Phase 1.

---

# 5. Item Size & Cost Efficiency

### Articles
Expected size: **5KB–30KB**  
Very efficient in DynamoDB.

### Tag index rows
Small: **1–2KB max**  
Extremely low read/write cost.

### GSIs
All GSIs are selective and designed for repeated read-heavy usage.

---

# 6. Query Patterns Covered

| Operation | Supported By |
|----------|----------------|
| Homepage latest | `GSI_AllArticles` |
| Category page | `GSI_ByType` |
| Tag search | `GSI_TagsIndex` |
| Article page | `PK = article_id` |
| Next/prev navigation | `published_at` values |
| Sidebar widgets | reusing type & tag queries |

Everything in the frontend is supported by these three indexes.

---

# 7. Future-Proofing

This schema supports:
- multi-language expansion (`language` field)
- separate region editions (if Brazil gets its own site)
- user accounts
- comments 
- polls
- stats & analytics
- caching improvements

Nothing will require breaking changes.

---

# 8. Related Documents

- `01-system-overview.md`
- `03-api-gateway-design.md`
- `04-lambda-functions.md`

---

*End of Document*