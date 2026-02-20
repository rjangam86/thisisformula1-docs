# 20 — Project Testing Checklist  
_ThisIsFormulaOne Platform — End-to-End Quality Assurance_

This document defines the complete testing & QA checklist for the platform.  
It merges **your previous Testing Plan** + **all missing improvements** into a single unified Phase-1 testing guide.

---

# 1. Testing Objectives

- Validate that all API endpoints return correct JSON envelopes  
- Confirm pagination logic across homepage, categories, tags  
- Ensure pages load dynamically with real DynamoDB content  
- Verify CloudFront caching + performance behaviour  
- Confirm SEO metadata generation  
- Validate mobile responsiveness  
- Ensure all security controls function correctly  
- Prepare the platform for continuous regression testing  

---

# 2. Testing Scope

### Included in Phase 1
- Homepage + featured article  
- Article page  
- Category pages (breaking / latest / tech / analysis / race_preview / race_debrief / featured)  
- Tag pages (driver names, team names, topics)  
- Pagination  
- SEO metadata + OpenGraph  
- Google Analytics v4  
- Lambda read-only operations  
- CloudFront behaviour  

### Excluded (future)
- Login & comments  
- User registration  
- Polls  
- Write operations  
- Admin panel  
- Shop integration  

---

# 3. API Testing Checklist

Test using **Postman**, **browser**, and **curl**.

### 3.1 GET /v1/articles
Verify:
- Default page loads (page=1)  
- Custom page loads (page=2,3…)  
- Exactly **6 items** returned  
- First = `featured`, next 5 = previews  
- Correct JSON envelope  
- Metadata accuracy (total_pages, total_items)  

### 3.2 GET /v1/articles/{id}
Verify:
- Correct HTML content returned  
- Missing ID returns 404  
- HTML sanitized without stripping structure  
- Slug-based routing loads correct article  

### 3.3 GET /v1/articles/by-type/{type}
Test all types:
- breaking  
- latest  
- tech  
- analysis  
- race_preview  
- race_debrief  
- featured  

Verify:
- Only that type appears  
- Sorted by published_at descending  
- Pagination behaves correctly  

### 3.4 GET /v1/articles/by-tag/{tag}
Verify:
- Tag filter (ferrari, hamilton, mercedes, redbull…)  
- Correct pagination  
- “No results” returns empty list, not error  

### 3.5 GET /v1/search?q=keyword
Phase-1 search:
- Case-insensitive  
- Title + excerpt + tags are scanned  
- Limit to 20 results  
- Results sorted by relevancy (keyword in title > tag > excerpt)  

---

# 4. DynamoDB Testing

### 4.1 Structure Validation
- Table name = `F1Articles`  
- PK = `article_id`  
- `GSI_AllArticles` exists  
- `GSI_ByType` exists  
- Throughput mode correct (On-Demand recommended)  

### 4.2 Sample Data Validation
Insert at least 20 realistic articles:
- Mix of types  
- Mix of tags  
- Range of timestamps  

### 4.3 Edge Case Testing
- Empty query results  
- Missing fields (excerpt, tags)  
- Extremely large content fields  
- Invalid data types  

---

# 5. Lambda Read-API Testing

### 5.1 Input Validation
- Invalid page numbers  
- Invalid type values  
- Invalid tags  
- Missing params default correctly  

### 5.2 DynamoDB Query Behaviour
- Query GSI_AllArticles  
- Query GSI_ByType  
- Scan+Filter for tag search  
- Pagination tokens handled without errors  

### 5.3 Fail Case Handling
Simulate:
- DynamoDB throttling  
- Lambda timeout  
- Invalid JSON responses  
- Missing article_id  

Ensure:
- Proper structured error format  
- Error logs contain request_id + timestamp  

---

# 6. Frontend Testing

### 6.1 Homepage
- Featured article displays correctly  
- Sidebar widgets load  
- Latest articles carousel scrolls  
- Pagination button at bottom works  

### 6.2 Article Page
- H1 title displays  
- Main image loads  
- HTML content renders safely  
- Tags section loads  
- SEO `<title>` updates dynamically  
- URL follows correct slug  

### 6.3 Category Pages
- Only articles of selected type load  
- Pagination works  
- Sidebar remains consistent  

### 6.4 Tag Pages
- Driver/team/topic filtering works  
- Pagination works  
- At least 3 common tags tested  

### 6.5 Mobile Testing
Test on:
- iPhone XR  
- iPhone 14  
- iPhone SE  
- Samsung Galaxy S22  
- iPad  
- Small Android devices  

Verify:
- Menu collapses  
- Font sizes correct  
- Image grid resizes  
- No horizontal scrolling (no overflow)  

---

# 7. SEO Testing

### 7.1 Meta Tags
- `<title>` correct  
- `<meta description>` correct  
- Canonical link correct  
- `<meta keywords>` generated  
- `<meta robots>` correct  

### 7.2 OpenGraph / Twitter Cards
Verify for all platforms:
- Facebook  
- Instagram Threads  
- X/Twitter  
- WhatsApp  
- LinkedIn  

Check:
- og:image  
- og:title  
- og:description  
- twitter:card  

### 7.3 Sitemap Validation
- `/sitemap.xml` loads  
- `/sitemap-articles.xml` loads  
- Validate via Google Search Console  

### 7.4 Structured Data (JSON-LD)
Test with:
- Google Rich Results Test  

---

# 8. CloudFront Testing

### 8.1 Caching
- API always dynamic (no cache)  
- Static assets cached (1 day)  
- Images cached (1 day or longer)  
- Category/tag pages cached (5 min)  

### 8.2 Global Distribution
Verify load time from:
- Brazil  
- India  
- US  
- Europe  
- Australia  

Goal:
- < 2 seconds global  
- < 1.2 seconds local  

### 8.3 Error Pages
- 404 page loads  
- 500 fallback loads  

---

# 9. Security Testing

### 9.1 CORS
Allowed:
- thisisformulaone.net  
- localhost  

Blocked:
- Unknown domains  
- Direct IP calls  

### 9.2 XSS Protection
Test input:
```
<script>alert('x')</script>
```
Should be sanitized by DOMPurify.

### 9.3 CSP (Content Security Policy)
Validate using:
- https://csp-evaluator.withgoogle.com  

### 9.4 Rate Limits
Perform:
- 50 rapid GET requests  
Expected:
- Throttling kicks in  

---

# 10. Performance Benchmarks

### Web
- Homepage < 1.2 sec  
- Article page < 1.3 sec  

### API
- Average < 250 ms  
- P95 < 400 ms  

### Images
- Optimised and lazy-loaded  

---

# 11. Pre-Launch Master Checklist

- [ ] Insert 30–50 articles  
- [ ] Load test key API endpoints  
- [ ] Confirm SEO titles for all pages  
- [ ] Verify GA4 is receiving traffic  
- [ ] Validate all category pages  
- [ ] Validate all tag pages  
- [ ] Test 20+ article slugs  
- [ ] Monitor logs for 24 hours  
- [ ] Confirm CloudFront caching rules  
- [ ] Run Lighthouse tests (mobile & desktop)  
- [ ] Document all known issues  

---

# 12. Related Documents

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
- `11-search-and-seo.md`  
- `12-comments-and-user-interaction.md`  
- `13-search-system.md`  
- `14-admin-panel.md`  
- `15-cms-design.md`  
- `16-shop-integration-blueprint.md`  
- `17-shop-security-checklist.md`  
- `18-seo-opengraph.md`  
- `19-data-maintenance.md`  

---