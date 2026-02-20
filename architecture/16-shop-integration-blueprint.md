# 16 — Shop Integration Blueprint  
_ThisIsFormulaOne Platform — E-Commerce Architecture Plan_

This document defines the blueprint for integrating an online shop into the ThisIsFormulaOne platform.  
Although the shop is **Phase 3**, the architecture must be planned now so the website, API, and CMS remain compatible.

The shop is expected to run via **Shopify + Printify**, with optional migration to a custom backend in the future.

---

# 1. Shop Vision

The shop will sell:

- T-shirts (drivers, teams, slogans, radios)  
- Hoodies / caps  
- Stickers  
- Limited editions  
- Event-based merch (Brazil GP, Monaco GP, etc.)

Key goals:

- Smooth integration into existing website  
- Fast browsing  
- Secure checkout  
- Easy update of products via Shopify dashboard  
- No duplication of inventory logic

---

# 2. Architecture Overview

| Component | Role |
|----------|------|
| **Shopify Storefront** | Handles all product data & checkout |
| **Shopify Storefront API** | Pulls product data into website |
| **Lambda — Shop Proxy** | Optional cache layer to speed the site |
| **CloudFront CDN** | Caching for product images + API calls |
| **Frontend Website** | Displays shop pages using your F1 theme |
| **Printify** | Handles production & fulfillment |
| **Stripe/Shopify Payments** | Payment processing (no card details on your server) |

Everything remains **serverless** on your side.

---

# 3. Shop URL Structure

Recommended:

```
https://thisisformulaone.net/shop
https://thisisformulaone.net/shop/product/{product_slug}
https://thisisformulaone.net/shop/cart (redirects to Shopify)
```

All checkout is done **on Shopify** (safer and simpler).

---

# 4. Shop Frontend Layout

## A. Main Shop Page
- Hero banner  
- Category selector (T-shirts, Hoodies, Caps)  
- Grid of products  
- Sorting (latest, price low→high)  

## B. Product Page
- Large product image  
- Carousel of images  
- Price  
- Description  
- Size selector  
- “Buy on Shopify” button  
- Related products (optional)

## C. Header Links
- Shop in navbar  
- Cart icon (optional, can redirect to Shopify)

---

# 5. Data Flow

### 1️⃣ Frontend calls your Lambda:
```
GET /v1/shop/products
```

### 2️⃣ Lambda calls Shopify Storefront API:
- Fetch products  
- Transform JSON (clean fields)  
- Cache result in CloudFront  

### 3️⃣ Frontend receives preformatted JSON:
```
{
  "products": [
    {
      "product_id": "123",
      "title": "Lando Norris LN4 Orange Tee",
      "price": "29.99",
      "image_url": "https://cdn.shopify.com/...",
      "slug": "ln4-orange-tee"
    }
  ]
}
```

---

# 6. API Endpoints (Read-Only)

### GET `/v1/shop/products`
- Returns all products  
- Supports pagination  
- Supports filtering by category  

### GET `/v1/shop/products/{slug}`
- Returns single product  
- Includes variants (sizes)

Shop endpoints are **read-only** and **public**.

---

# 7. DynamoDB Usage (Optional)

You may store:

- Product metadata  
- Click statistics  
- Shop analytics  
- Stock caching (only metadata, not inventory)

But **do not store prices or checkout data** — Shopify owns that.

---

# 8. Syncing & Caching Strategy

### Caching Rules
- Cache `/v1/shop/products` for 10 minutes  
- Cache product detail for 30 minutes  
- Invalidate automatically when:
  - Product changes on Shopify  
  - New product added  

### Pre-rendering future:
- Lambda can store recent product lists in S3 JSON file  
- CloudFront serves JSON instantly  
- Lambda updates S3 file every 10 minutes  

---

# 9. Security Requirements

- **Never** store credit card details  
- **Never** process checkout on your backend  
- All checkout must redirect to Shopify’s secure checkout  
- Use strict CORS: only allow your domain  
- Protect admin/shop configuration endpoints with IAM only  
- Rotate Shopify API token frequently  

---

# 10. CMS Integration

Shop is not part of CMS.  
But CMS may need:

- “Featured product of the week”  
- “Add product to article page”  
- “Shop CTA in article”

Data is pulled from Shopify into CMS.

---

# 11. Future Shop Extensions

- Personalized recommendations  
- “Shop the look” linked under articles  
- Driver/team category pages  
- F1-themed bundles  
- AI tools to generate shop images  
- Printify order tracking widget in user profile  
- Multi-currency support (AUD, USD, BRL, INR)  

---

# 12. Summary

This blueprint defines a clean, scalable integration between ThisIsFormulaOne.net and Shopify.  
The shop remains a standalone system but integrates visually and structurally with the main site.

The platform stays:

- Serverless  
- High-performance  
- Secure  
- Easy to expand