# 19 — Data Maintenance  
_ThisIsFormulaOne Platform — Long-Term Content Integrity, Cleanup & Lifecycle Rules_

This document describes all **data retention**, **cleanup**, **archival**, and **consistency** rules for the platform.  
It ensures the DynamoDB tables, API responses, cached objects, and image assets remain healthy as the system grows.

---

# 1. Objectives

Data maintenance ensures:

- Articles remain consistent and searchable  
- Database size stays controlled (cost-efficient)  
- Orphaned images and unused slugs are cleaned  
- Caches remain accurate  
- Future migrations become easier (v2 API, new tables, search index)

This is a **backend-only document** and affects Lambdas, DynamoDB, and S3.

---

# 2. Maintenance Responsibilities

| Component | Responsibility |
|----------|----------------|
| **DynamoDB** | Cleanup stale drafts, fix broken slugs, ensure indexes remain hot |
| **S3 Image Bucket** | Remove unused/orphan images |
| **Search Index (future)** | Rebuild keyword index when article updated |
| **CloudFront Cache** | Purge outdated pages |
| **API Layer** | Validate article integrity before serving |

---

# 3. Article States & Lifecycle

### 3.1 Article States
Each article will store:

```
status: "PUBLISHED" | "DRAFT" | "ARCHIVED"
```

- **PUBLISHED** → visible on homepage, feeds, SEO, sitemaps  
- **DRAFT** → not shown on public API  
- **ARCHIVED** → still accessible via `/articles/slug` but hidden from lists

### 3.2 Lifecycle Rules

| State | TTL / Action |
|-------|--------------|
| **Draft** | Auto-delete after **30 days** if not published |
| **Published** | Kept indefinitely |
| **Archived** | Kept indefinitely but removed from category/tag listings |
| **Deleted** | Hard-deleted from DB + S3 image cleanup |

---

# 4. DynamoDB Maintenance Tasks

### 4.1 Remove Expired Drafts

A scheduled Lambda runs **daily**:

- Query GSI on `status = "DRAFT"`  
- Filter items older than 30 days  
- Delete them  
- Collect image URLs → send to S3 cleanup job

---

### 4.2 Validate Required Fields

A weekly Lambda scans published articles to check:

- Missing **slug**  
- Missing **image_url**  
- Missing **excerpt**  
- Missing **article_type**  
- Duplicate slugs  
- Incorrect timestamps  
- Tags improperly formatted (should be lowercase array)

Report → saved to `maintenance_reports` S3 folder.

---

### 4.3 Normalize Timestamps

If any article has:

- `published_at` in the future  
- or missing timestamp  

Fix or flag them.

---

### 4.4 Cold Data Archival (Phase 2)

Older articles (> 3 years) may be moved to an **Archive Table**:

```
F1Articles_Archive
```

Purpose:

- Reduce main table size  
- Keep search fast  
- Still allow SEO access via article slug

---

# 5. S3 Image Maintenance

Two S3 folders:

```
/images/articles/{id}.jpg
/images/temp/{uuid}.jpg
```

### 5.1 Remove Temp Files

- Auto-delete objects older than **7 days** in `/temp/`  
- Handled by S3 Lifecycle Policy (no Lambda needed)

---

### 5.2 Remove Orphaned Images

The system detects:

- Images in S3 that are **not referenced** by any DynamoDB article

A monthly Lambda:

- Lists all objects in `/articles/`  
- Builds a set of all `image_url` references from DB  
- Compares  
- Deletes orphans  
- Generates a report

---

# 6. Slug & URL Integrity Maintenance

Every slug must remain **unique** and **stable**.

Maintenance Lambda checks:

- Duplicate slugs  
- Accidental slug changes (wrong updates)  
- Slugs not matching naming convention:
  - lowercase
  - no spaces
  - hyphens only

If incorrect → auto-fix or report.

---

# 7. Tag Integrity & Cleanup

Rules:

- Tags stored in lowercase  
- Tags without articles are cleaned  
- Tags must not contain spaces (use hyphens)

A bi-weekly process:

- Scan table  
- Build tag frequency map  
- Remove ghost tags  
- Regenerate `sitemap-tags.xml`

---

# 8. Index Maintenance (GSI)

### 8.1 Check GSI Health

For each GSI (AllArticles, ByType):

- Validate read capacity consumption  
- Check for hot partitions  
- Ensure keys are uniform (e.g., article_type correctly distributed)

### 8.2 Reindexing (Phase 2)

When Phase-2 search index is added:

- Re-generate FTS index for all articles  
- Remove outdated tokens  
- Compact index storage

---

# 9. Caching Maintenance

### 9.1 CloudFront Invalidation Rules

When an article is updated:

- Invalidate:
  - `/articles/{slug}`
  - `/v1/articles/*` (API JSON)
  - Category pages (e.g., `/category/breaking`)
  - Tag pages (e.g., `/tag/lewishamilton`)

A background Lambda performs invalidations every 10–30 minutes in batch.

### 9.2 Daily Content Cold Cache Purge

Optional daily job:

- Purge cached API JSON so DynamoDB can refresh trending content

---

# 10. Automated Maintenance Schedule

| Task | Frequency | Component |
|------|-----------|-----------|
| Draft cleanup | Daily | DynamoDB |
| Missing fields validator | Weekly | DynamoDB |
| Slug integrity checker | Weekly | DynamoDB |
| Tag frequency analyzer | Bi-weekly | DynamoDB |
| Orphaned image cleanup | Monthly | S3 |
| GSI health check | Monthly | DynamoDB |
| Sitemap regeneration | Daily | Lambda + S3 |
| CloudFront cache batch invalidation | Every 10–30 min | CloudFront |

---

# 11. Manual Maintenance Tools (Admin API)

Not exposed publicly.

```
POST /admin/maintenance/run-draft-cleanup
POST /admin/maintenance/run-slug-check
POST /admin/maintenance/normalize-tags
POST /admin/maintenance/rebuild-sitemap
```

Accessible only from the Admin Panel.

---

# 12. Related Documents

- `02-dynamodb-schema.md`
- `03-api-gateway-design.md`
- `04-lambda-read-api.md`
- `07-site-structure.md`
- `13-search-system.md`
- `18-seo-opengraph.md`
- `20-project-testing-checklist.md`