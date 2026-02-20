# 13 — Site Security  
_ThisIsFormulaOne Platform — Web & API Security Guidelines_

This document defines security controls for the **frontend**, **API Gateway**, **Lambda**, **CloudFront**, and **AWS infrastructure**.  
Even though Phase 1 is *read-only*, security must be prepared for future login, comments, and user accounts.

---

# 1. Security Goals

The platform must ensure:

1. **Safe public access** (no sensitive endpoints exposed)
2. **Protection against bots, scrapers, DDoS**
3. **Protection against injection attacks**
4. **Secure API access from website only**
5. **Hardened AWS permissions**
6. **Future-proofing for authentication**

---

# 2. Frontend Security

### 2.1 HTTPS Only  
Enforced via:

- CloudFront HTTPS Redirect
- HSTS enabled (minimum age 6 months)

### 2.2 No sensitive user data stored in browser  
LocalStorage and sessionStorage must never contain:

- Passwords  
- Tokens  
- Personal info  

### 2.3 Sanitized HTML Rendering  
Article `content` might include:

- `<p>`, `<b>`, `<i>`, `<h1>`  
- Links  
- Line breaks  

Rules:

- No `<script>` allowed  
- No inline JS  
- No iframes (Phase 1)  
- Sanitize all incoming HTML fragments

Suggested client library: **DOMPurify**.

---

# 3. API Gateway Security

### 3.1 Public Read-Only API  
All API routes are GET-only.

No mutation endpoints exist in Phase 1.

### 3.2 CORS Enforcement  
Allowed origins:

```
https://thisisformulaone.net
http://localhost:3000
```

Blocked:

- `*` wildcard  
- Any unknown domain

### 3.3 Throttling  
Suggested:

- 200 req/sec (burst)
- 1000 req/min (sustained)

This protects from traffic spikes and abuse.

### 3.4 WAF (Optional for Phase 2)

Rules to add later:

- SQL injection block  
- XSS block  
- Bot management  
- Rate limiting per IP (custom rule)  

---

# 4. Lambda Security

### 4.1 Least Privilege IAM  
Each Lambda receives:

```
dynamodb:GetItem
dynamodb:Query
logs:CreateLogStream
logs:PutLogEvents
```

NOT ALLOWED:

- dynamodb:DeleteItem  
- dynamodb:PutItem  
- s3:PutObject  
- sns:*  
- sqs:*  
- secretsmanager:*  

### 4.2 No environment secrets  
Only static config allowed:

- TABLE_NAME  
- ENVIRONMENT  
- REGION  

Secrets stay in Secrets Manager (Phase 2 when login is added).

### 4.3 Input Validation  
All request parameters must be validated:

- `page` must be 1–500  
- `per_page` must be 1–20  
- `type` must match known article types  
- `tag` must be alphanumeric + hyphens  

Reject anything else.

---

# 5. DynamoDB Security

### 5.1 No public access  
Table must be accessible only from the read-only Lambda role.

### 5.2 Data Integrity  
Articles are inserted only through internal automation (not this API).  
API cannot modify anything.

### 5.3 Protected GSIs  
GSIs must inherit table permissions — no additional exposure.

---

# 6. CloudFront Security

### 6.1 Origin Shield  
Enabled (Sydney region) to mitigate DDoS and improve caching.

### 6.2 Caching Rules  
API requests bypass caching where required:

- `/v1/articles/*` → short TTL  
- Static assets (`/assets/*`) → long TTL

### 6.3 Blocked HTTP Methods  
Allow only:

- GET  
- HEAD  

Block:

- POST  
- PUT  
- DELETE  
- OPTIONS (handled by CORS preflight)

---

# 7. Attack Prevention

### 7.1 SQL Injection  
Impossible — DynamoDB is NoSQL, but sanitization still required.

### 7.2 Cross-Site Scripting (XSS)  
Mitigation:

- Sanitize article HTML with DOMPurify  
- Escape template variables in frontend  
- Enforce Content Security Policy (CSP)

Suggested CSP:

```
default-src 'self';
script-src 'self' https://*.googletagmanager.com;
img-src 'self' https://s3.amazonaws.com https://*.thisisformulaone.net data:;
style-src 'self' 'unsafe-inline';
```

### 7.3 Directory Traversal  
Blocked automatically because no filesystem access exists.

### 7.4 DDoS  
Handled by CloudFront + throttling + (optional) WAF.

---

# 8. Future: User Accounts & Authentication

Phase 2 will introduce:

### 8.1 Cognito User Pools  
Authentication for:

- Commenting  
- User profiles  
- Likes, bookmarks  
- User preferences

### 8.2 Secure Comment API  
Endpoints:

```
POST /v1/articles/{id}/comments
GET  /v1/articles/{id}/comments
```

Requires JWT validation via Cognito Authorizer.

### 8.3 Rate Limiting  
One comment every 15 seconds per user.

### 8.4 Moderation Pipeline  
- Auto-filter hate speech  
- Auto-detect spam  
- Manual moderation queue

(Not required for Phase 1.)

---

# 9. Related Documents

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
- `12-monitoring-and-analytics.md`

---