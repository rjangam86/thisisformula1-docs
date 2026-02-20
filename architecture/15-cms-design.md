# 15 — CMS Design  
_ThisIsFormulaOne Platform — Content Management System Blueprint_

This document defines the architecture, workflow, and data structures for the CMS (Content Management System) powering the editorial pipeline.  
The CMS is the backend engine used by the Admin Panel (File 14) for managing all articles, drafts, images, metadata, tags, and publishing flows.

---

# 1. Purpose of the CMS

The CMS provides:

- Article creation & editing  
- Draft storage  
- Metadata management  
- Image library integration  
- Tag/driver/team management  
- Publishing workflow  
- Revision history (future)  
- Multi-language support (future)

The CMS is purely backend, exposed via Admin API endpoints.  
It does **not** power the public site directly — the public site reads via `/v1/articles` endpoints.

---

# 2. CMS Architecture Overview

| Component | Role |
|----------|------|
| **Admin API (REST)** | Main CMS interface |
| **Lambda CMS Handlers** | Process CRUD operations |
| **DynamoDB (F1Articles)** | Store article data |
| **S3 Bucket** | Store article images |
| **CloudWatch** | Logs, alerts |
| **Cognito (future)** | User auth |
| **Admin Frontend** | CMS UI |

The CMS is designed to be fully serverless.

---

# 3. Article Lifecycle

## A. Draft Creation
- Admin user creates a draft (status = "DRAFT")
- Minimal required fields: `title`, `excerpt`, `article_type`
- CMS assigns `article_id` (UUID)
- CMS stores draft in DynamoDB

## B. Editing Drafts
- Admin updates content, tags, drivers, teams, images
- CMS validates fields and writes updates

## C. Publishing
- Status → `"PUBLISHED"`
- Adds item to GSIs for feed sorting
- Available immediately through public API

## D. Retiring / Archiving
- Status → `"RETIRED"`
- Hidden from public feeds
- Still accessible to Admin

## E. Revision History (Phase 3)
- CMS will store previous versions in `/revisions` subtable

---

# 4. CMS Database Model (DynamoDB)

The CMS writes to the `F1Articles` table with the following structure:

```
article_id (PK)
title
slug
excerpt
content (HTML)
image_url
image_source
article_source
article_type
tags (list)
drivers (list)
teams (list)
published_at (number)
created_at (number)
updated_at (number)
status ("DRAFT" | "PUBLISHED" | "RETIRED")
language ("en" | "pt-br")
```

Indexes used:

- **GSI_AllArticles** → global feed  
- **GSI_ByType** → category pages  
- **GSI_Tags** → tag pages  
- **Future: GSI_ByDriver / GSI_ByTeam** (if needed)

---

# 5. CMS API Endpoints

These endpoints are internal-only, consumed by the Admin Panel.

### Article CRUD  
| Method | Route | Description |
|--------|--------|-------------|
| POST | `/admin/v1/articles` | Create draft |
| GET | `/admin/v1/articles/{id}` | Fetch article |
| PUT | `/admin/v1/articles/{id}` | Update article |
| DELETE | `/admin/v1/articles/{id}` | Delete/retire article |
| GET | `/admin/v1/articles?status=draft` | List drafts |
| GET | `/admin/v1/articles?status=published` | List published |

### Tag + Metadata  
| Method | Route | Description |
|--------|--------|-------------|
| GET | `/admin/v1/tags` | List tags |
| POST | `/admin/v1/tags` | Add tag |
| DELETE | `/admin/v1/tags/{tag}` | Remove tag |

### Images  
| Method | Route | Description |
|--------|--------|-------------|
| POST | `/admin/v1/images` | Upload image |
| GET | `/admin/v1/images` | List images |
| DELETE | `/admin/v1/images/{id}` | Delete image |

---

# 6. CMS Validation Rules

## Required Fields
- `title`
- `article_type`
- `excerpt`
- `content` (required only when publishing)

## Auto-generated Fields
- `article_id`
- `slug`
- `created_at`
- `updated_at`

## Allowed Types
- `"breaking"`  
- `"analysis"`  
- `"tech"`  
- `"results"`  
- `"feature"`  
- `"preview"`  
- `"debrief"`  

You can extend this list easily.

---

# 7. Slug Generation Rules

Slugs are generated automatically:

```
"Leclerc wins in Japan!" → "leclerc-wins-in-japan"
```

Rules:

- Lowercase  
- Replace spaces with hyphens  
- Remove punctuation  
- Limit 80 chars  
- Ensure uniqueness by appending `-2`, `-3` if needed  

---

# 8. Image Integration

Admin user uploads image → CMS validates:

- File type  
- File size  
- Dimensions  
- MIME type  

Stored in:

```
s3://thisisformulaone-media/articles/<article_id>/<filename>.jpg
```

Returned to Admin Panel as CDN URL.

---

# 9. Publishing Workflow Requirements

When publishing:

- Validate all required fields  
- `published_at = current_epoch`  
- Write item to DynamoDB  
- Invalidate CloudFront cache for:
  - homepage  
  - tag page  
  - type page  
  - article page

(Cache invalidation will be handled by Deployment Lambda.)

---

# 10. Multilingual Support (Phase 3)

CMS must support:

- English source article  
- Auto-translated PT-BR version  
- Ability to edit translations manually  
- Option to publish only one or both languages  
- Store translations as separate records:
  ```
  article_id: abc123-en
  article_id: abc123-pt
  ```

---

# 11. Permissions & Security

- Only Admin/CMS roles allowed  
- All admin traffic HTTPS enforced  
- No public write API  
- Validate all HTML inputs (prevent XSS)  
- Do not log article bodies  
- Only signed URLs for image uploads (future)  

---

# 12. Future: CMS Extensions

- Scheduler for timed articles  
- AI-assisted drafting (via your scraper → paraphraser → CMS)  
- Bulk tag editing  
- Bulk post importing (PlanetF1, RN365)  
- Version history system  
- Approval flows (Editor → Publisher → Live)  

---

# 13. Summary

The CMS is the **editorial engine** controlling how articles are created, stored, edited, and published.  
It integrates tightly with:

- Admin Panel  
- DynamoDB  
- Image system  
- Public API  

It is fully expandable and future-proof for multilingual, analytics, polls, comments, and shop integration.