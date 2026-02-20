# 09 — Caching & Performance Strategy  
_ThisIsFormulaOne Platform — CDN, Browser, API, and Lambda Performance Rules_

This document defines the full caching architecture for frontend assets, CDN images, API responses, and Lambda compute.  
It ensures the site loads **fast worldwide** with **minimal AWS cost**.

---

# 1. Caching Overview

The system uses **four layers of caching**:

1. **Browser Cache**  
2. **CloudFront CDN Cache**  
3. **API Response Cache (CloudFront + Lambda)**  
4. **Database Query Optimization**

Each layer works independently but stacks for maximum speed.

---

# 2. Browser Caching Rules

Static assets:

| File Type | Cache TTL |
|-----------|-----------|
| `.css` | 30 days |
| `.js` | 30 days |
| `.woff2` | 1 year |
| `.jpg`, `.png`, `.webp` | 30 days |
| `.html` | No cache (always revalidate) |

Browser headers (sent by S3/CloudFront):

```
Cache-Control: public, max-age=2592000, immutable
```

HTML:

```
Cache-Control: no-cache, no-store, must-revalidate
```

---

# 3. CloudFront CDN Caching

CDN domain:

```
https://cdn.thisisformulaone.net
```

Caching behavior:

| Path | TTL | Notes |
|------|-----|-------|
| `/images/*` | 24h | Cached aggressively |
| `/js/*` | 30d | Fingerprinted in production |
| `/css/*` | 30d | Fingerprinted in production |
| `/articles/*` | 10m | Future-proof for dynamic pages |
| `/v1/articles*` (API) | 15s – 30s | API caching layer |

Default: **Origin Shield enabled**

GZIP + Brotli compression on.

---

# 4. API Caching Strategy (Very Important)

API base:

```
https://api.thisisformulaone.net/v1
```

API caching is performed entirely via **CloudFront**, not Lambda.

### TTL rules:

| Endpoint | TTL | Reason |
|----------|------|---------|
| `GET /articles?page=*` | 20s | Homepage refreshes often |
| `GET /articles/{id}` | 60s | Detail pages stable |
| `GET /articles/by-type/*` | 20s | Like homepage sections |
| `GET /articles/by-tag/*` | 20s | Tag clusters frequently hit |

### Important:
Lambda returns the header:

```
Cache-Control: max-age=20
```

But CloudFront overrides TTL if required.

---

# 5. API Query Optimization (DynamoDB)

Guidelines:

- Homepage always queries **GSI_AllArticles**  
- Category pages query **GSI_ByType**  
- Tag pages use **Scan + Filter** (Phase 2: Inverted Index table)

To minimize DynamoDB cost:

1. Always **limit page sizes**  
2. Prefer **KeyConditionExpression** instead of filters  
3. Enable **CloudFront caching** to avoid repeated Lambda calls  

---

# 6. Lambda Caching / Cold Start Mitigation

We configure:

- **Node.js 22.x** (fast cold starts)  
- **512 MB Memory** (optimal for DynamoDB Query)  
- **Provisioned Concurrency (optional)**  
  - Only during race weekends  
  - Saves 80% of cold starts  

### Lambda response includes:

```
Cache-Control: max-age=20, public
```

CloudFront respects it.

---

# 7. HTML Page Caching Strategy

HTML pages are static:

- `index.html` → No cache, but leverage CDN  
- `category.html` → No cache  
- `article.html` → No cache  
- `tag.html` → No cache  
- `search.html` → No cache  

Reason:
All content is driven by API, not HTML markup.

---

# 8. Static Assets Fingerprinting

To avoid browser caching older JS/CSS:

```
main.4f28c1.js
styles.8da9e2.css
```

Build process auto-inserts correct filenames.

Rather than invalidating CloudFront everyday,
fingerprinting ensures updates propagate instantly.

---

# 9. Real-Time Widgets (Phase 2)

Live widgets (e.g., race results) will use:

- `Cache-Control: max-age=5`
- CloudFront TTL override 3–5 seconds
- Optional WebSockets for near real-time

Not required for Phase 1.

---

# 10. Purge Strategy (When Needed)

You almost NEVER purge the whole CDN.

Only purge:

- One article image  
- One JS/CSS file  
- One page section  

Commands:

```
aws cloudfront create-invalidation \
  --distribution-id XXXXXX \
  --paths "/images/articles/1763*.jpg"
```

Avoid global `/*` invalidations — expensive and unnecessary.

---

# 11. Summary Table

| Layer | Purpose | TTL |
|--------|----------|------|
| Browser | Speed on repeat visits | 30 days |
| CloudFront (Static) | Global asset delivery | 24h–30d |
| CloudFront (API) | Reduces Lambda cost | 20–60s |
| Lambda | Fresh data | Executes only when needed |
| DynamoDB | Primary store | Highly optimized |

---

# 12. API Rate Limiting & Throttling (API Gateway)

This section defines the rate-limiting and throttling policy for all `/v1/*` REST API endpoints.

---

## 12.1 Why Rate Limiting Is Required

- Protects Lambda cost from abusive traffic  
- Prevents API Gateway cold starts from sudden spikes  
- Avoids DynamoDB hot partitions  
- Limits scraping and unwanted bots  
- Stabilizes global performance across CloudFront  

Rate limiting ensures the API remains fast and predictable.

---

## 12.2 API Gateway Default Limits (Per Stage)

| Setting | Value | Notes |
|--------|--------|-------|
| **Burst Limit** | 100 req/sec | Short-term spike tolerance |
| **Steady Rate** | 50 req/sec | Long-term allowed rate |
| **Timeout** | 29 sec | Lambda limit |
| **Integration retries** | Enabled | For transient failures |

These values are **safe defaults** for a news website.

---

## 12.3 Custom Rules for Public Endpoints

### GET /v1/articles  
- Steady: **40 req/sec**  
- Burst: **80 req/sec**  
- Cached by CloudFront: YES  
- High-traffic homepage protection  

### GET /v1/articles/{id}  
- Steady: **30 req/sec**  
- Burst: **60 req/sec**  
- Not cached (dynamic)  
- Protects DynamoDB requests per-article  

### GET /v1/articles/by-type/{type}  
- Steady: **25 req/sec**  
- Burst: **50 req/sec**  
- Cached for 5 minutes via CloudFront  

### GET /v1/articles/by-tag/{tag}  
- Steady: **20 req/sec**  
- Burst: **40 req/sec**  
- DynamoDB Scan+Filter → MUST RATE LIMIT  

---

## 12.4 Geo-Based Throttling (Optional)

CloudFront edge nodes can enforce soft throttling:

| Region | Limit | Reason |
|-------|--------|--------|
| India | 2x traffic | High audience volume |
| Brazil | 2x traffic | High audience volume |
| Europe | normal | Balanced load |
| Australia | normal | Close to your origin |

This protects Lambda costs during global spikes.

---

## 12.5 Anti-Bot Protection

Add API Gateway rules:

- Block User-Agents with `bot`, `scraper`, `crawler` unless known good  
- IP rate limiting per 5 minutes window  
- Challenge suspicious patterns (repeated requests for old pages)  

Optional CloudFront Functions:

- Lightweight JS challenge  
- Signature-based request filtering  

---

## 12.6 When Throttling Activates

User will receive:

```
HTTP 429 Too Many Requests
Retry-After: 1
```

Lambda will **not execute** in this case → saves money.

---

## 12.7 Logging Throttled Events

CloudWatch Log Fields:

- `source_ip`
- `user_agent`
- `endpoint`
- `timestamp`
- `request_id`
- `rate_limit_triggered: true`

Helps detect abuse patterns.

---

## 12.8 Recommended Alarms (CloudWatch)

- **TooManyRequests** > 100 in 1 minute  
- **5XXError** > 5 in 1 minute  
- DynamoDB `ConsumedReadCapacity` > projected  

---

## 12.9 Future Enhancements (Phase 2)

- Token-based API access for certain partners  
- Per-IP quotas  
- Signed URLs for heavy endpoints  
- Dynamic WAF Rules (AWS WAF Bot Control)  

---

# 13. Related Documents

- `01-overview.md`  
- `02-dynamodb-schema.md`  
- `03-api-gateway-design.md`  
- `04-lambda-read-api.md`  
- `05-frontend-integration.md`  
- `06-auth-and-users.md`  
- `07-site-structure.md`  
- `08-image-handling.md`

---