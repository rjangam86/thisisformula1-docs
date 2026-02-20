# 04 — Lambda Read API  
_ThisIsFormulaOne Platform — Read-Only Backend Logic_

This document defines the Lambda functions that power all public API read operations.  
All endpoints from API Gateway `/v1/*` will invoke these Lambdas using **Lambda Proxy Integration**.

---

## 1. Overview

| Property | Value |
|---------|--------|
| **Runtime** | Node.js 22.x |
| **Type** | Read-only Lambdas (no writes) |
| **Input** | API Gateway Proxy Event |
| **Output** | JSON envelope (`data`, `meta`) |
| **DB** | DynamoDB (F1Articles + GSIs) |
| **Error Format** | Standard API error JSON |

All Lambdas follow the same structure and share utility functions across a common library (Phase 2 cleanup).

---

## 2. Lambda Functions List

| Lambda Name | Purpose |
|-------------|----------|
| **GetArticlesFeed** | Homepage feed + pagination |
| **GetArticleById** | Single full article |
| **GetArticlesByType** | Category pages (breaking, tech, preview…) |
| **GetArticlesByTag** | Tag pages (Ferrari, Hamilton, etc.) |
| **SearchArticles** | Full-text search (Phase 2) |

Each function runs independently but shares the same environment variables.

---

## 3. Shared Environment Variables

| Variable Name | Purpose |
|----------------|----------|
| `ARTICLES_TABLE` | DynamoDB Table: `F1Articles` |
| `GSI_ALL_ARTICLES` | Index for homepage feed |
| `GSI_BY_TYPE` | Index for article_type queries |
| `DEFAULT_PAGE_SIZE` | Usually 6 |
| `MAX_PAGE_SIZE` | Usually 12 |

---

## 4. Standard Response Format

All Lambdas return:

```json
{
  "data": {},
  "meta": {
    "page": 1,
    "per_page": 6,
    "total_pages": 15,
    "total_items": 88
  }
}
```

This is strictly enforced for consistency.

---

## 5. Lambda: GetArticlesFeed  
### Route  
`GET /v1/articles?page=1&per_page=6&type=&tag=`

### Logic
1. Read `page`, `per_page`, `type`, `tag`.
2. If `type` exists → redirect to Type Lambda.  
3. If `tag` exists → redirect to Tag Lambda.  
4. Otherwise → query `GSI_AllArticles`:
   - `PK = "PUBLISHED"`
   - `SK = published_at DESC`
5. Compute pagination:
   - featured = first item  
   - previews = next 5 items  
6. Return envelope.

---

## 6. Lambda: GetArticleById  
### Route  
`GET /v1/articles/{article_id}`

### Logic
1. Get `article_id` from path.
2. DynamoDB `GetItem` on primary table.
3. If not found → return 404.
4. Return full article object.

---

## 7. Lambda: GetArticlesByType  
### Route  
`GET /v1/articles/by-type/{type}?page=1&per_page=6`

### Logic
1. Query `GSI_ByType`:
   - `PK = article_type`
   - `SK = published_at DESC`
2. Paginate and return:
   - featured = first item
   - previews = next 5

Types used:
- breaking  
- tech  
- analysis  
- preview  
- results  
- race_debrief  
- general (fallback)

---

## 8. Lambda: GetArticlesByTag  
### Route  
`GET /v1/articles/by-tag/{tag}?page=1&per_page=6`

### Logic
1. Query `GSI_AllArticles`:
   - `PK = "PUBLISHED"`
2. FilterExpression:
   - `contains(tags, :tag)`
3. Paginate and return.

Phase 2 will replace this with the `F1TagIndex` table.

---

## 9. Lambda: SearchArticles (Phase 2)  
### Route  
`GET /v1/search?q=ferrari&limit=10`

### Logic
- Uses DynamoDB Scan + filters OR  
- OpenSearch Lite cluster later (optional).  

Not implemented in Phase 1.

---

## 10. Pagination Logic (All Lambdas)

- Default per_page: 6  
- Page 1 returns:
  - `data.featured = items[0]`
  - `data.previews = items[1..5]`
- Next pages return:
  - No featured
  - All items returned as previews
- Meta block always included.

---

## 11. Error Handling Pattern

All errors return:

```json
{
  "error": {
    "message": "Not found",
    "code": 404
  }
}
```

No HTML errors.  
No internal AWS stack traces exposed.

---

## 12. Performance & Caching

### CloudFront:
- Cache GET responses for 30–120 seconds.
- Invalidate on new article publish.

### DynamoDB:
- Queries only hit DB for uncached requests.
- Data size remains minimal (<1 MB per item).

---

## 13. Related Documents
- `02-dynamodb-schema.md`
- `03-api-gateway-design.md`
- `05-frontend-integration.md`
- `07-site-structure.md`

---