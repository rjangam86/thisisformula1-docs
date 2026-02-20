# 17 — Shop Security Checklist  
_ThisIsFormulaOne Platform — E-Commerce Security Requirements_

This document defines all mandatory and recommended security measures for integrating the Shopify + Printify shop into the ThisIsFormulaOne platform.  
Since the shop processes real customer payments, security must be strict even though the checkout itself is handled externally by Shopify.

---

# 1. Core Principles

- Your platform **must not store or process any payment data**.  
- Checkout must always happen on **Shopify’s PCI-compliant** system.  
- The website must protect customer interactions, product browsing, and admin access.  
- API tokens and secrets must be protected with **IAM + Secrets Manager**.  

---

# 2. Required Security Controls

## 2.1 CORS Restrictions
Only allow:

```
https://thisisformulaone.net
http://localhost:3000   (development)
```

Never use `"*"` in production.

---

## 2.2 Shopify API Token Security
- Store Shopify Storefront API tokens in **AWS Secrets Manager**.  
- Access allowed ONLY from:
  - Shop Lambda function (read-only).  
  - Admin CMS Lambda (future).  

- Rotate token every **90 days**.

---

## 2.3 Lambda Function Security
- IAM policy must be limited to:
  - `secretsmanager:GetSecretValue`
  - **NO** DynamoDB write permissions for shop.
  - CloudWatch logging only.

- Use:
  - Environment variable whitelist  
  - No inline secrets  
  - Minimum dependency footprint

---

## 2.4 CloudFront + S3 Security
- Block all S3 public access except via CloudFront Origin Access Control.  
- Enable CloudFront security features:
  - HTTPS only  
  - HTTP/2  
  - TLS 1.2 or above  
  - Signed URLs for admin shop previews (optional)

---

## 2.5 API Gateway Security
- Public endpoints must use **read-only** Lambda handlers.  
- Admin endpoints (future CMS) must:
  - Require IAM or Cognito auth  
  - Block external access  
  - Log every request  

---

# 3. Shopify Security Responsibilities

These are handled by Shopify and do NOT require backend implementation:

- PCI DSS Compliance  
- Secure checkout  
- Fraud detection & risk analysis  
- Secure card vault storage  
- GDPR compliance  
- Data encryption at rest  

You must **never** attempt to duplicate these functions.

---

# 4. Frontend Security Requirements

## 4.1 No client-side secrets
The frontend must NOT contain:
- Shopify token  
- Admin URLs  
- Hidden CMS endpoints  

Frontend only receives **clean JSON** from your Lambda.

---

## 4.2 Sanitise HTML & product descriptions
Shopify product descriptions can contain HTML.  
Your frontend must sanitise:

- `<script>`  
- Inline JS  
- Event handlers (e.g., `onclick`)  

to prevent XSS.

---

## 4.3 Content Security Policy (CSP)
Add a CSP to the site:

```
default-src 'self';
img-src 'self' https://cdn.shopify.com https://*.cloudfront.net data:;
script-src 'self';
style-src 'self' 'unsafe-inline';
connect-src 'self' https://api.thisisformulaone.net;
frame-src https://shopify.com https://*.shopify.com;
```

---

# 5. Admin & CMS Security (Future)

While not implemented yet, the architecture must allow:

- Admin login (Cognito or IAM Identity Center)
- Product metadata editing (tags, featured, banners)
- Server-side action logs
- Multi-factor authentication (MFA) for all admin accounts

These will be covered in the **admin-panel.md** and **cms-design.md** documents.

---

# 6. Monitoring & Alerting

Use CloudWatch to create alerts for:

- Increased 4xx/5xx errors  
- Spikes in shop Lambda invocations  
- Unusual API error patterns  
- Secret rotation failures  
- CORS violation logs  

Additionally enable:

- AWS GuardDuty  
- AWS WAF (future, optional)  

---

# 7. Summary

This checklist ensures the ThisIsFormulaOne shop integration is:

- Secure  
- PCI-compliant (through Shopify)  
- Scalable  
- Easy to maintain  
- Safe for customers and your business  

The shop integration is **frontend-heavy + Shopify-powered**, while your backend remains serverless and low-risk.