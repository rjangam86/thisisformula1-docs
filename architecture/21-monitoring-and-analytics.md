# 12 — Monitoring & Analytics  
_ThisIsFormulaOne Platform — Traffic, Telemetry & Health Monitoring_

This document defines **Google Analytics integration**, **backend telemetry**, **Lambda logging standards**, **CloudWatch monitoring**, and **error-tracking strategy** for the platform.

---

# 1. Goals

Monitoring and analytics must answer four questions:

1. **Is the website getting traffic?**
2. **How many people read each article?**
3. **Are backend systems healthy?**
4. **Can we detect errors instantly?**

This design covers all four.

---

# 2. Google Analytics (GA4) Integration

GA4 is used to track:

- Pageviews
- Article reads
- Navigation flows
- Traffic sources
- Browser & device data
- Engagement time
- Bounce rate

Integration strategy:

### 2.1 GA Tag Script
Inject into `/index.html`:

```html
<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-XXXXXXX"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());
  gtag('config', 'G-XXXXXXX');
</script>
```

Replace `G-XXXXXXX` with your actual Measurement ID.

---

# 3. Per-Article Analytics

For each article page:

```
gtag('event', 'view_article', {
  article_id: '12345',
  title: 'Leclerc signs new Ferrari contract',
  category: 'breaking',
  tags: ['leclerc', 'ferrari'],
});
```

Benefits:

- Shows which articles are popular
- Helps predict future content performance
- Feeds your automation pipeline (Phase 2, recommendations)

---

# 4. API Analytics & Logging

All read-only Lambdas follow a unified logging structure.

### 4.1 Log Format (Every Request)

```
[API] GET /v1/articles?page=1&type=latest
RequestId: a7fbd123
Params: { page:1, type:'latest' }
DurationMs: 28
Results: 6 items
```

### 4.2 Error Log Format

```
[ERROR] /v1/articles/by-tag/ferrari
RequestId: a7fbd123
ErrorType: DynamoDB.QueryException
Message: ...
```

All logs go to:

```
/aws/lambda/ThisIsF1_ReadAPI
```

---

# 5. CloudWatch Dashboards

CloudWatch Dashboard: `ThisIsF1-Website-Metrics`

Graphs include:

- Lambda Invocations
- Lambda Duration (avg, p95)
- Lambda Errors
- DynamoDB Read Capacity Usage
- API Gateway 5xx Errors
- API Gateway Latency
- CloudFront Cache Hit Ratio

Dashboard is updated automatically once configured (AWS-managed metrics).

---

# 6. CloudWatch Alarms

Recommended alarms:

| Alarm | Trigger | Action |
|-------|---------|--------|
| **Lambda Errors** | > 5 errors in 5 min | SNS notification to your email |
| **DynamoDB Throttles** | > 1 throttle in 2 min | Increase capacity |
| **API 5xx** | > 2 errors in 5 min | SNS alert |
| **CloudFront Low HIT Ratio** | < 60% for 10 min | Investigate caching rules |

SNS Topic Example:

```
arn:aws:sns:ap-southeast-2:XXXXXXXXX:ThisIsF1_Alerts
```

---

# 7. Frontend Health Monitoring

Add a lightweight heartbeat endpoint:

```
GET /v1/health
```

Response:

```json
{ "status": "ok", "timestamp": 1763522200 }
```

CloudWatch alarm:

- If the health check fails 3 times → trigger alert.

Optional: Ping every 10 minutes via Lambda to ensure availability.

---

# 8. Error Tracking (Phase 2)

Once the site grows, integrate:

- **Sentry**
- or **LogRocket**

Tracks:

- JS frontend errors
- Broken pages
- Session replays
- Rage clicks / UI bugs

Not required for Phase 1.

---

# 9. Performance Metrics

You must track:

| Metric | Goal |
|--------|------|
| Google Lighthouse Performance | **> 90** |
| Google Lighthouse SEO | **> 95** |
| Page load time | **< 1.5 sec** (cached) |
| First Contentful Paint | **< 0.8 sec** |
| Time to Interactive | **< 1.2 sec** |

Tools:

- Google PageSpeed Insights
- Chrome DevTools Lighthouse
- CloudFront Reports

---

# 10. Backend Observability Plan

### 10.1 Structured Logs
All Lambda logs must use **JSON** for machine parsing.

### 10.2 Metrics Filters
Examples:

- Count tag searches
- Count by-type page loads
- Count article pageviews (API side)

### 10.3 SLA Monitoring
Trigger alarms for:

- > 1000 requests/min spike (DDoS-like)
- Latency > 300ms p95 for 3 min

---

# 11. Future Extensions (Phase 2)

- Real-time analytics dashboard (CloudWatch → QuickSight)
- Search analytics (top queries, no-result queries)
- Article performance scoring
- Viewer heatmaps

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

---