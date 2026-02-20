# 01 — System Overview & Architecture  
_ThisIsFormulaOne Platform (F1 News Website + AWS Backend)_

## 1. Introduction
This document provides a full high-level overview of the entire Formula 1 news platform, including:

- Frontend website (static HTML/JS served via S3 + CloudFront)
- Backend APIs (AWS API Gateway + Lambda)
- Database (DynamoDB)
- Image assets workflow
- Future features (comments, users, polls)
- Architectural decisions and constraints

This serves as the **master reference** before going into deeper details in other documentation files.

---

## 2. High-Level Architecture Components

### 2.1 Frontend (Static + Dynamic Fetching)
- Hosted on **S3** with CloudFront CDN.
- HTML/CSS/JS template from MinBeriMag (sports variant).
- jQuery or vanilla JS used to load articles dynamically.
- Page types:
  - Homepage (with multiple sections: breaking, featured, latest, tech)
  - Article page
  - Category pages: Breaking News, Analysis, Race Preview, Results, Tech
  - Contact page
  - Shop (placeholder for future integration)
- Dynamic sections fed via **REST API**.

---

## 3. Backend API
### 3.1 Technology Stack
- AWS **API Gateway (REST API)**  
- AWS **Lambda (Node.js 22)**  
- AWS **DynamoDB** (primary article store)  
- AWS **CloudFront** (caching layer)

### 3.2 API Characteristics
- Fully **read-only** for Phase 1.
- Public access (no authentication).
- CORS restricted to:
  - https://thisisformulaone.net
  - localhost (for dev)
- Versioned under `/v1/`.

### 3.3 Core Endpoints (Phase 1)
- `GET /v1/articles`  
  Paginated article feed with featured + previews.

- `GET /v1/articles/{id}`  
  Returns full article for the article page.

- `GET /v1/articles/by-tag/{tag}`  
  (Tag search / hashtag indexing)

- `GET /v1/articles/by-type/{type}`  
  Supports category pages:
  - breaking
  - latest
  - analysis
  - tech
  - race_preview
  - results

### 3.4 Future Endpoints
- `/v1/comments/*`
- `/v1/users/*`
- `/v1/polls/*` (deferred)

These will be documented but not implemented initially.

---

## 4. DynamoDB Overview
A single table design simplifies the architecture and querying model.

Table: **F1Articles**

Core attributes:
- `article_id` (PK)
- `title`
- `content` (full HTML)
- `excerpt`
- `image_url`
- `image_source`
- `article_source`
- `published_at` (epoch)
- `article_type`
- `tags` (list)
- `slug`

Indexes:

### GSI 1 — `GSI_AllArticles`
Used for latest news pagination.
- PK: `status` (static: "PUBLISHED")
- SK: `published_at`

### GSI 2 — `GSI_ByType`
Used for category pages.
- PK: `article_type`
- SK: `published_at`

### GSI 3 — `GSI_TagsIndex`
Used for hashtag search.
- PK: `tag_name`
- SK: `published_at`

---
  
## 5. Article Workflow (End-to-End)
### 5.1 Article Creation (Automated via your existing pipeline)
1. Article scraped → processed → paraphrased → translated → structured.
2. Image selected → stored in S3.
3. Final article JSON stored in DynamoDB.
4. API immediately reflects it.

### 5.2 Rendering on Website
1. User visits homepage.
2. jQuery fetches:

https://api.thisisformulaone.net/v1/articles?page=1

3. Featured article shown first.
4. Remaining 5 previews shown in widgets.
5. Sections (breaking, tech, etc.) fetch via type filters.
6. Clicking article = new page fetch via slug or ID.

---

## 6. Frontend Architecture Summary
### 6.1 Pages
- `/index.html` → homepage  
- `/article.html?id={id}` → full article  
- `/type.html?type={article_type}` → category page  
- `/tags.html?tag={tag}` → hashtag page  
- `/contact.html`  
- `/shop.html` (disabled for now)

### 6.2 Dynamic Loaders
Each dynamic page will include a JS loader:
- `loadHomepage()`
- `loadArticle(id)`
- `loadByType(type)`
- `loadByTag(tag)`

These will be written later.

---

## 7. User System (Future)
Not implemented now, but architecture planned:

### 7.1 DynamoDB: F1Users
- `user_id`
- `email`
- `password_hash`
- `created_at`

### 7.2 Login Flow (Future)
- Cognito or custom JWT
- Required for commenting later

---

## 8. Comment System (Future)
Not implemented now, but planned.

### Table: F1Comments
- `comment_id`
- `article_id`
- `user_id`
- `text`
- `created_at`
- `status` (approved/pending)

### API (Future)
- `POST /v1/comments`
- `GET /v1/comments/{article_id}`
- `DELETE /v1/comments/{comment_id}`

---

## 9. Poll System (Deferred)
Feature documented but not built now.

### Table: F1Polls
- poll_id
- question
- options
- votes

---

## 10. Performance + Caching
- CloudFront caching for:
- `/v1/articles`
- all type filters
- all tag filters

- DynamoDB TTL for old preview records (optional)

---

## 11. Security Notes
- No write APIs exposed publicly.
- IAM policies restrict Lambdas to specific tables and GSIs.
- No secret keys on frontend.
- S3 buckets locked to CloudFront Origin.

---

## 12. Implementation Roadmap (High-Level)
1. Create DynamoDB table + GSIs  
2. Implement `/v1/articles` Lambda  
3. Implement `/v1/articles/{id}` Lambda  
4. Implement `/v1/articles/by-type/{type}`  
5. Implement `/v1/articles/by-tag/{tag}`  
6. Frontend integration with homepage  
7. Frontend integration with article page  
8. Category pages  
9. Tag pages  
10. SEO + OpenGraph metadata  
11. Extend into comments + users (future)  
12. Analytics integration (Google Analytics)

---

## 13. Related Files in This Documentation
- `02-dynamodb-schema.md`
- `03-api-gateway-design.md`
- `04-lambda-functions.md`
- `05-frontend-integration.md`
- `06-routing-navigation.md`
- `07-future-features.md`
- `08-implementation-roadmap.md`

---

*End of Document*