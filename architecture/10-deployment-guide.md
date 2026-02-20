# 10 — Deployment Guide  
_ThisIsFormulaOne Platform — API, Lambda, CloudFront & Domain Deployment Instructions_

This document describes exactly how to deploy the **REST API**, **Lambda functions**, **DynamoDB table**, **CloudFront CDN**, and **custom domain routing**, step-by-step.  
It is written so YOU execute all AWS setup manually — **no automated provisioning**.

---

# 1. Prerequisites

You must have:

- AWS Account with AdminAccess  
- Domain purchased (thisisformulaone.net)  
- S3 bucket for frontend hosting  
- S3 media bucket (`tif1-media`)  
- DynamoDB Table (`F1Articles`)  
- At least one Lambda created (Read API Lambda)

AWS CLI installed (optional but helpful).

---

# 2. API Gateway Deployment (REST API)

## 2.1 Create the REST API

AWS Console → API Gateway → **Create API**

- Choose: **REST API**
- Build: **New API**
- Name: `ThisIsF1API`
- Endpoint Type: Edge-Optimized (CloudFront integration)
- Create

---

## 2.2 Create Resources & Routes

Create the base resource `/v1`:

```
/v1
    /articles
    /articles/{article_id}
    /articles/by-type/{type}
    /articles/by-tag/{tag}
```

For each route:

### Method → GET  
Integration → **Lambda Proxy Integration**

Select your Lambda:  
`ThisIsF1_ReadAPI_Lambda`

Enable:

✔ Use Lambda Proxy Integration  
✔ Use Default Timeout (29 sec)

Save.

---

## 2.3 Enable CORS

For `/v1/*`:

Allowed Origins:

```
https://thisisformulaone.net
http://localhost:3000
```

Allowed Methods:

```
GET
```

Allowed Headers:

```
Content-Type
```

Max Age:

```
86400
```

---

## 2.4 Deploy the API

Create a Stage:

- Name: **prod**
- Stage URL will be:  
  `https://{api-id}.execute-api.{region}.amazonaws.com/prod/v1/`

---

# 3. Connect API Gateway to Custom Domain

AWS Console → API Gateway → Custom Domain Names

Create Domain:

```
api.thisisformulaone.net
```

TLS Certificate (ACM) in **us-east-1**:

```
api.thisisformulaone.net
```

Mapping:

| Path | Target Stage |
|------|--------------|
| /v1 | prod          |

Route53:

```
CNAME api.thisisformulaone.net → target from API Gateway
```

Test:

```
https://api.thisisformulaone.net/v1/articles
```

---

# 4. Deploy the Read Lambda

Lambda Name:

```
ThisIsF1_ReadAPI_Lambda
```

Runtime:

```
Node.js 22.x
```

Handler file:

```
index.js
```

Environment Variables:

| Key | Value |
|-----|-------|
| TABLE_NAME | F1Articles |
| CDN_BASE | https://cdn.thisisformulaone.net |
| REGION | ap-southeast-2 (or SA-east-1 for BR) |
| LOG_LEVEL | info |

Permissions:

✔ `dynamodb:Query` on the table  
✔ `dynamodb:GetItem`  
✔ CloudWatch Logs  

Set Memory:

```
512 MB
```

Timeout:

```
10 seconds
```

---

# 5. DynamoDB Table Requirements

Table Name:

```
F1Articles
```

Partition Key:

```
article_id (String)
```

Secondary Indexes:

### GSI #1 — For Homepage

```
GSI_AllArticles:
  PK: status = "PUBLISHED"
  SK: published_at (Number)
```

### GSI #2 — For Category Pages

```
GSI_ByType:
  PK: article_type
  SK: published_at
```

Capacity Mode: On-Demand (recommended)

---

# 6. Frontend Deployment (Website)

## 6.1 Create S3 Bucket

```
thisisformulaone-website
```

Properties:
- Static Website Hosting: Enabled
- Index document: `index.html`

Upload:

- `index.html`
- `/css/*`
- `/js/*`
- `/images/*`

---

## 6.2 CloudFront Distribution

Origin:

```
S3 → thisisformulaone-website
```

Settings:

| Setting | Value |
|--------|--------|
| Allowed HTTP | Redirect HTTP → HTTPS |
| TTL | 0 for HTML |
| TTL | 30 days for JS/CSS |
| TTL | 24 hours for images |
| Compress | GZIP + Brotli |

Alternate Domain:

```
thisisformulaone.net
www.thisisformulaone.net
```

SSL Certificate in ACM (us-east-1):

```
thisisformulaone.net
*.thisisformulaone.net
```

---

## 6.3 Route53 DNS Setup

```
A (Alias) → CloudFront Distribution
```

Test:

```
https://thisisformulaone.net
```

---

# 7. Media CDN (Images)

Use the existing CloudFront distribution:

```
cdn.thisisformulaone.net
```

Origin:

```
tif1-media/images
```

Everything in the articles table MUST reference:

```
https://cdn.thisisformulaone.net/images/articles/{id}.jpg
```

---

# 8. Versioning Strategy

Backend:

```
/v1/articles
/v1/articles/{id}
```

When breaking changes occur:

```
/v2/articles
/v2/articles/{id}
```

Frontend references API version explicitly.

---

# 9. Mandatory Testing Procedure

### 9.1 API Tests

```
curl https://api.thisisformulaone.net/v1/articles
curl https://api.thisisformulaone.net/v1/articles/1763550009
curl https://api.thisisformulaone.net/v1/articles/by-type/tech
curl https://api.thisisformulaone.net/v1/articles/by-tag/leclerc
```

### 9.2 Website Tests

- Homepage loads correctly  
- Category pages load  
- Article detail loads  
- Pagination works  
- All images load from CDN  
- No CORS errors  
- No mixed content errors  

---

# 10. Known Issues & Notes

- Tag Pages use scan + filter in Phase 1  
- Full-text search requires a new table (Phase 2)  
- User login + comments will extend API Gateway with POST routes  
- If API becomes slow → enable CloudFront caching on API routes  

---

# 11. Related Documents

- `01-overview.md`  
- `02-dynamodb-schema.md`  
- `03-api-gateway-design.md`  
- `04-lambda-read-api.md`  
- `05-frontend-integration.md`  
- `06-auth-and-users.md`  
- `07-site-structure.md`  
- `08-image-handling.md`  
- `09-caching-strategy.md`

---