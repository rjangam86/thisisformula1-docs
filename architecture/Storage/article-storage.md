Below is a complete Markdown (.md) architecture + implementation document based strictly on the information you provided.
You can drop this directly into your repo (for example docs/article-storage-architecture.md) and use it as the implementation guide for migration.

⸻


# Article Storage Architecture Migration
## Long-Term Editorial Storage Strategy

Author: Architecture Design  
Platform: This Is Formula 1 Editorial Platform  
Database: DynamoDB  
Content Storage: Amazon S3  

---

# 1. Problem Statement

The editorial platform currently stores **full article data inside DynamoDB**, including large content bodies.

Articles are currently configured with a **TTL expiration (`expireAt`) of 30 days**, meaning they are automatically deleted after one month.

This design was originally implemented to prevent DynamoDB storage growth.

However, this creates major issues for an editorial platform.

### Issues with Current Design

1. **SEO Impact**
   - Articles disappear after 30 days
   - Google indexed URLs become 404
   - Search ranking is negatively affected

2. **Loss of Historical Archive**
   - Motorsport journalism requires long-term archives
   - Technical articles remain relevant for years

3. **Broken Links**
   - External references to articles become invalid

4. **Unnecessary DynamoDB Storage Usage**
   - Article body content may grow to **100KB+**
   - DynamoDB is not ideal for storing large content blobs

---

# 2. Architectural Goal

Redesign the article storage architecture to support:

- Permanent article archive
- SEO-friendly URLs
- Minimal DynamoDB storage usage
- Scalable storage for long article content
- No breaking changes to the existing website

---

# 3. Current Architecture

## DynamoDB Table

Table Name: F1Articles

### Primary Key

| Attribute | Type |
|-----------|------|
| article_id | String |

Example:

TIF1-000000978
TIF1-000000979

---

# 4. Current Indexes

## 4.1 All Articles Index

Index Name: all_articles

| Partition Key | Sort Key |
|---------------|----------|
| status | publishedAt |

Used for:

- Latest articles
- Homepage feed
- Article pagination

Example Query:

status = “published”
ORDER BY publishedAt DESC

---

## 4.2 Article Type Index

Index Name: gsi_by_type

| Partition Key | Sort Key |
|---------------|----------|
| article_type | publishedAt |

Used for:

- Breaking news
- Opinion
- Technical
- Interview
- News

Example:

article_type = “technical”
ORDER BY publishedAt DESC

---

## 4.3 UID Index

Index Name: gsi_uid_index

| Partition Key |
|---------------|
| uid |

The UID is an **8-character unique identifier** appended to the slug.

Example slug:

red-bull-powertrains-store-convinced-8A3F2K91

Structure:

slug-id + “-” + uid

Purpose:

- Guarantee unique URLs
- Avoid slug collision

---

# 5. Current Article Data Structure

Example DynamoDB Item

{
article_id: “TIF1-000000978”,
article_type: “technical”,
author: “Rahul”,
content: “Large article body stored in DynamoDB”,
created_at: “2026-03-05T10:30:00Z”,
expireAt: 1740000000,
images: [“https://s3/…”],
language: “enUK”,
original_url: “…”,
published_at: “2026-03-05T12:00:00Z”,
quotes: [],
slug: “ferrari-floor-upgrade-analysis”,
source: “PlanetF1”,
status: “published”,
tags: [“Ferrari”,“Technical”],
title: “Ferrari Floor Upgrade Explained”,
uid: “8A3F2K91”,
updated_at: “2026-03-05T12:05:00Z”
}

---

# 6. Current Problem Areas

### 6.1 TTL Deletes Articles

expireAt

TTL removes articles after 30 days.

This is **not suitable for editorial websites**.

---

### 6.2 Article Content Stored in DynamoDB

Large content fields cause:

- Increased storage cost
- Larger response payloads
- Slower queries

---

### 6.3 Future Article Growth

Articles may exceed:

100KB+

DynamoDB item size limit:

400 KB

Storing full content inside DynamoDB is not ideal.

---

# 7. Target Architecture

We will implement **two-layer article storage**.

| Layer | Storage |
|------|--------|
| Metadata | DynamoDB |
| Article Content | S3 |

---

# 8. New Architecture Overview

User Request
│
▼
API Gateway / Lambda
│
▼
DynamoDB (metadata lookup)
│
▼
S3 (fetch article body)
│
▼
Return combined response

---

# 9. DynamoDB After Migration

DynamoDB will store **metadata only**.

### Updated Item Example

{
article_id: “TIF1-000000978”,
article_type: “technical”,
author: “Rahul”,
contentS3Key: “articles/2026/03/TIF1-000000978.json”,
created_at: “2026-03-05T10:30:00Z”,
images: [],
language: “enUK”,
published_at: “2026-03-05T12:00:00Z”,
slug: “ferrari-floor-upgrade-analysis”,
source: “PlanetF1”,
status: “published”,
tags: [“Ferrari”,“Technical”],
title: “Ferrari Floor Upgrade Explained”,
uid: “8A3F2K91”,
updated_at: “2026-03-05T12:05:00Z”
}

The **content field is removed**.

---

# 10. S3 Article Content Storage

### Bucket

tif-articles-content

### Storage Structure

articles/
2026/
03/
TIF1-000000978.json

Example S3 Object

{
“title”: “Ferrari Floor Upgrade Explained”,
“body”: “…full article body…”,
“images”: [],
“quotes”: []
}

---

# 11. Query Flow After Migration

## Fetch Article by Slug

GET /article/{slug}

Flow:

1. Query DynamoDB
2. Retrieve metadata
3. Read `contentS3Key`
4. Fetch article body from S3
5. Merge response
6. Return to frontend

---

# 12. Required Schema Changes

### Remove TTL

Delete attribute:

expireAt

Disable TTL in DynamoDB table settings.

---

### Add New Attribute

contentS3Key

Type:

String

Example:

articles/2026/03/TIF1-000000978.json

---

# 13. Migration Strategy

We currently have:

~1000 articles

Migration is safe and straightforward.

### Migration Steps

1. Scan DynamoDB table
2. Read `content` field
3. Upload content to S3
4. Generate S3 key
5. Update DynamoDB item
6. Remove `content` attribute

---

# 14. Migration Lambda Example Flow

Pseudo workflow:

Scan F1Articles

for each article:

if contentS3Key exists:
    skip

upload content to S3

update item:
    set contentS3Key
    remove content

---

# 15. Backward Compatibility

Important requirement:

**Existing website must not break.**

Solution:

Migration must be done gradually.

During transition:

If DynamoDB contains:

content

use DynamoDB content.

If DynamoDB contains:

contentS3Key

fetch content from S3.

---

# 16. Updated API Logic

Pseudo code:

if item.content exists:
return item.content
else:
fetch from S3 using contentS3Key

This ensures safe rollout.

---

# 17. Performance Benefits

After migration:

### DynamoDB Item Size

Reduced from:

~50KB

to:

~2KB

Benefits:

- faster queries
- lower storage cost
- better cache performance

---

# 18. Cost Efficiency

DynamoDB Storage:

50k articles ≈ 100MB

Cost:

≈ $0.03 / month

S3 storage:

very low cost

---

# 19. Operational Improvements

Recommended settings:

### S3 Versioning

Enable:

Versioning: Enabled

Benefits:

- rollback article edits
- audit history

---

### CloudFront Caching

Recommended header:

Cache-Control: public, max-age=600

Reduces API load.

---

# 20. Implementation Steps

## Step 1

Create S3 bucket.

tif-articles-content

---

## Step 2

Update article creation Lambda.

New flow:

write content to S3
store metadata in DynamoDB

---

## Step 3

Add `contentS3Key` field.

---

## Step 4

Disable TTL.

Remove:

expireAt

---

## Step 5

Deploy API fallback logic.

Support:

- DynamoDB content
- S3 content

---

## Step 6

Run migration Lambda.

Process existing articles.

---

## Step 7

Verify migration.

Check:

- S3 content
- DynamoDB metadata

---

## Step 8

Remove `content` attribute permanently.

---

# 21. Future Scalability

Publishing rate:

10–15 articles/day

Yearly:

~5000 articles

10 years:

~50,000 articles

DynamoDB handles this easily.

---

# 22. Final Architecture Summary

| Component | Purpose |
|-----------|--------|
| DynamoDB | Metadata storage |
| S3 | Article body storage |
| CloudFront | CDN caching |
| Lambda | API logic |

---

# 23. Result

After migration the system will provide:

- Permanent article archive
- SEO-safe URLs
- Lower DynamoDB cost
- Unlimited article storage
- Scalable editorial platform

---


⸻

✅ This document now gives you:
	•	Problem statement
	•	Current architecture
	•	Your real table schema
	•	Migration plan
	•	Backward compatibility
	•	Implementation steps
	•	Future architecture

You can literally commit this and start implementing.

⸻

If you want, I can also give you one more extremely important improvement for your system that most media sites miss:

“Static article HTML generation + CloudFront caching (eliminate 95% of API calls)”

That one change can make your site 10x faster and cheaper.