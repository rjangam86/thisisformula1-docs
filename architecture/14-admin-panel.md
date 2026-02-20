# 14 — Admin Panel  
_ThisIsFormulaOne Platform — Internal Management Interface_

This document defines the architecture, UI structure, API requirements, and security policies for the **Admin Panel** — the internal tool used to publish/manage articles, handle drafts, edit metadata, and view platform analytics.

The Admin Panel is **Phase 2**, but its design must be compatible with all existing backend architecture from Files 01–13.

---

# 1. Purpose of the Admin Panel

The Admin Panel allows **internal staff only** to:

- Create new articles  
- Manage drafts  
- Edit/update article content  
- Delete/retire posts  
- View article metadata (views, tags, status)  
- Upload/select images  
- View system alerts (failed crawls, missing images, etc.)  
- Moderate comments (future module)  
- Manage users (future module)  
- View analytics (future module)

**The Admin Panel DOES NOT affect the public site.**  
It is a separate restricted web app + API.

---

# 2. High-Level Architecture

| Component | Role |
|----------|------|
| **Admin Frontend** | React or lightweight HTML/JS dashboard |
| **Admin API Gateway (private)** | Separate from public API |
| **Admin Lambdas** | CRUD operations for content |
| **DynamoDB (F1Articles)** | Stores article content + metadata |
| **S3** | Image uploads (admin-only) |
| **CloudFront (admin disabled)** | No caching for admin |
| **Cognito (future)** | Admin users + auth |

The Admin API uses **IAM or Cognito authentication**, never public access.

---

# 3. Admin URL Structure

Recommended:

- Admin frontend: `https://admin.thisisformulaone.net`
- Admin API base: `https://api.thisisformulaone.net/admin/v1`

All admin paths begin with `/admin/v1/*`.

---

# 4. Admin Panel Features (Phase 2 + Expandable)

## Core Features (Phase 2)
- Create Article  
- Edit Article  
- Delete Article  
- Manage Drafts  
- Generate Slug  
- Set article type (breaking, tech, analysis, results, etc.)  
- Set tags  
- Set driver/team cross-reference  
- Upload/select images  
- Preview article before publishing  

## Future Features
- Comment moderation  
- User management  
- Poll creation  
- AI-assisted article drafts  
- Scheduled publishing  
- Analytics dashboard (views, clicks, engagement)  
- Multi-language publishing (EN + PT-BR)

---

# 5. Admin Panel UI Layout

### Suggested Layout
- **Left Sidebar (Navigation)**
  - Dashboard  
  - All Articles  
  - Drafts  
  - Create New Article  
  - Tags  
  - Images  
  - Settings  
  - Comments (future)  
  - Users (future)

- **Top Bar**
  - Search  
  - Profile / Logout  

- **Main Panel**
  - Table/List view  
  - Pagination  
  - Filters (type, date, tag, status)  
  - Button: "Create New Article"

---

# 6. Admin API Endpoints

All Admin endpoints are private and require auth.

### Article Management
| Method | Route | Description |
|--------|--------|-------------|
| **POST** | `/admin/v1/articles` | Create article |
| **PUT** | `/admin/v1/articles/{id}` | Update article |
| **DELETE** | `/admin/v1/articles/{id}` | Delete/retire |
| **GET** | `/admin/v1/articles` | List articles with filters |
| **GET** | `/admin/v1/articles/{id}` | Get full article data |

### Tag Management
| Method | Route | Description |
|--------|--------|-------------|
| **GET** | `/admin/v1/tags` | List all tags |
| **POST** | `/admin/v1/tags` | Create tag |
| **DELETE** | `/admin/v1/tags/{tag}` | Delete tag |

### Image Library
| Method | Route | Description |
|--------|--------|-------------|
| **POST** | `/admin/v1/images` | Upload image |
| **GET** | `/admin/v1/images` | List existing images |
| **DELETE** | `/admin/v1/images/{id}` | Delete image |

### Authentication (Future)
| Method | Route | Description |
|--------|--------|-------------|
| **POST** | `/admin/v1/auth/login` | Login via Cognito |
| **POST** | `/admin/v1/auth/logout` | Logout |

---

# 7. Access Control

Access is restricted:

- Only AWS Cognito Admin User Pool  
- IAM role for Lambda → DynamoDB writes  
- HTTPS only  
- No public access  
- No CloudFront caching  

Roles (future):

- **Admin** — full access  
- **Editor** — create/edit articles  
- **Publisher** — approve/publish  
- **Analyst** — read analytics  

---

# 8. DynamoDB Usage

The Admin API writes to the same table:

```
F1Articles
```

Admin Panel manages:

- article_id  
- title  
- slug  
- excerpt  
- content  
- image_url  
- image_source  
- article_source  
- article_type  
- tags  
- teams  
- drivers  
- published_at  
- created_at  
- updated_at  
- status (PUBLISHED / DRAFT / RETIRED)

---

# 9. Draft vs Published Flow

### Draft
- Stored in DynamoDB with status = "DRAFT"
- Not visible to public API
- Appears only in Admin Panel

### Publish
- status updated to "PUBLISHED"
- added to GSI indexes automatically
- visible in `/v1/articles` feeds

### Retire
- status = "RETIRED"
- removed from homepage feeds
- still accessible via admin

---

# 10. Image Workflow (Admin Only)

1. Admin uploads image → S3 bucket `/images/`  
2. Lambda validates:
   - MIME type  
   - Size  
   - Dimensions (optional)  
3. Returns CDN URL to Admin  
4. Admin associates image with an article  

---

# 11. Security Requirements

- No client-side secrets  
- Strict CORS:  
  - allow only `admin.thisisformulaone.net`
- All admin calls must be authenticated  
- Lambda logs MUST NOT contain article content  
- Use CloudWatch alarms for:
  - Unauthorized access  
  - Throttling  
  - High error rate  
- Enable WAF (Future Phase)  

---

# 12. Future: Admin Dashboard Widgets

- Article performance  
- Most viewed tags  
- Writer performance  
- Real-time content queue  
- Error logs from crawlers  
- Failed image uploads  
- Scheduled posts summary  

---

# 13. Summary

The Admin Panel is an **internal-only** interface used to manage all editorial content, metadata, images, and operations for ThisIsFormulaOne.net.  
It is future-proofed to support comments, analytics, shop integration, polls, AI-assisted content, and multi-language editing.