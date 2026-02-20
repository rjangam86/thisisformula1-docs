# 13 — Search System  
_ThisIsFormulaOne Platform — Article Search Architecture_

This document defines the **search architecture**, **API design**, **indexing strategy**, and **future enhancements** for enabling fast, scalable article search across the website.  
The search system must support keywords, tags, drivers, teams, and full-text search.

---

# 1. Search Requirements

The system must allow users to:

- Search by **keywords** in titles and content  
- Search by **driver** (e.g., “Hamilton”)  
- Search by **team** (e.g., “McLaren”)  
- Search by **tags** (Ferrari, Mercedes, Crash, Rumor…)  
- Search by article type/category  
- Return results sorted by relevance

Performance targets:

- P99 response time < 250ms (cached), < 500ms (uncached)  
- Pagination support  
- Scalable to > 50,000 articles  
- Must integrate with CloudFront caching where safe  

---

# 2. High-Level Architecture

| Component | Role |
|----------|------|
| **Frontend** | Sends search queries via `/v1/search?q=...` |
| **API Gateway (REST)** | Public read-only endpoint |
| **Lambda – Search Processor** | Executes DynamoDB/ElasticSearch queries |
| **DynamoDB** | Stores article metadata + tags |
| **Optional Future: OpenSearch** | Full-text search for large dataset |
| **CloudFront** | Caches popular queries |

---

# 3. Search Endpoint

## GET `/v1/search`

### Query Parameters

| Param | Type | Description |
|-------|------|-------------|
| `q` | string | Search text (required) |
| `page` | int | Pagination page (default 1) |
| `per_page` | int | Items per page (default 10) |
| `type` | string | Optional filter by article type |
| `tag` | string | Optional filter by tag |
| `driver` | string | Optional filter by driver |
| `team` | string | Optional filter by team |

---

# 4. Search API Response Envelope

```json
{
  "data": [
    {
      "article_id": "123",
      "title": "Hamilton domina no Brasil",
      "excerpt": "Lewis Hamilton brilhou no GP de São Paulo...",
      "image_url": "https://...",
      "article_type": "breaking",
      "tags": ["Hamilton", "Mercedes"],
      "published_at": 1763512301,
      "slug": "hamilton-brasil-domina"
    }
  ],
  "meta": {
    "q": "hamilton",
    "page": 1,
    "per_page": 10,
    "total_items": 42,
    "total_pages": 5
  }
}
```

---

# 5. DynamoDB Search Strategy (Phase 1)

Because DynamoDB is not a full-text search engine, search will use:

### A. Title/Excerpt "contains" filtering  
- A **begins_with** or **contains** filter is acceptable for early stage  
- Fast enough for ~20–50k rows with caching

### B. Tag-based queries  
Use a **GSI**:

```
GSI_Tags:
  PK: tag
  SK: published_at
```

This allows:

- `/v1/articles/by-tag/{tag}`
- `/v1/search?tag=Hamilton`

### C. Article type filter  
Handled via the existing GSI_ByType.

### D. Ranking  
Initial rule-based ranking:

1. Title match  
2. Tag match  
3. Content match (if implemented)  
4. Published date  

---

# 6. When to Switch to OpenSearch (Phase 3)

Upgrade to **OpenSearch** when:

- > 50k articles  
- Full-text search across article bodies required  
- Sorting by relevance becomes mandatory  
- Ranking model needed (TF-IDF/BM25)

Architecture with OpenSearch:

- Lambda → OpenSearch query → Return ranked results
- DynamoDB only stores metadata

---

# 7. Frontend Search Integration

### Search Bar Flow

1. User types into search bar  
2. JS triggers `/v1/search?q=...`  
3. Results displayed in a dedicated "Search Results" page  
4. Pagination shown at bottom  
5. Clicking result → full article page  

### Suggested UI Components

- Sticky search bar or floating icon  
- Live autocomplete (Phase 2)  
- Category chips (e.g., “Breaking”, “Tech”, “Ferrari”)  

---

# 8. Anti-Abuse & Rate Limiting

- Prevent bots hammering search endpoint  
- Apply API Gateway rate limits per IP  
- Cache common queries in CloudFront  
- Sanitize input to remove HTML/script payloads  

---

# 9. CloudFront Caching Strategy for Search

Since search is dynamic:

- Cache **only frequent queries**  
- TTL: 60 seconds  
- Use cache-key including `q`, `page`, `type`  

Example cache key:

```
/v1/search?q=hamilton&page=1&per_page=10&type=breaking
```

---

# 10. Future Enhancements

- Autocomplete suggestions  
- Trending searches  
- Synonym dictionary (e.g., “Charles” → “Leclerc”)  
- Multi-language search (EN + PT-BR)  
- AI result ranking  
- Personalized results based on user interests  

---

# 11. Summary

The search system begins with a **DynamoDB-first** approach using filters, GSIs, and rule-based ranking.  
It later evolves into **OpenSearch** when scalability demands increase.  
All components integrate seamlessly with your existing API architecture and frontend structure.