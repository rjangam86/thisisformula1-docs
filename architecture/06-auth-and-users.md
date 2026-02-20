# 06 — Auth & User System (Future Expansion)
_ThisIsFormulaOne Platform — User Accounts, Login, and Commenting Architecture_

Although **Phase 1 does NOT implement login or commenting**, this document defines the **future-proof architecture** so that the website can expand later *without breaking existing API or database design*.

This file is strictly *architecture only* — no implementation yet.

---

# 1. Goals

We want a user system that supports:

- Optional login (email/password)
- Optional social login (later)
- Commenting on articles
- Like/upvote system (later)
- User profile pages (optional)
- Admin/moderator roles (phase 3)

But none of this will be implemented now — only planned.

---

# 2. Auth Model

### **Authentication Type (later phase):**
- JSON Web Token (JWT)
- Managed via AWS Lambda
- DynamoDB-backed user accounts
- Secure password hashing (bcrypt)

### **Login Flow (future):**
1. User enters email/password  
2. Lambda verifies credentials  
3. Lambda returns JWT  
4. Frontend stores token in memory/localStorage  
5. Token attached to requests requiring authentication (comment posting)

For now → **No auth required for reading content**.

---

# 3. User Table (DynamoDB)

Table Name: **F1Users**  
Partition Key: `user_id` (UUID)

| Field | Type | Notes |
|-------|------|-------|
| user_id | String | Unique ID |
| email | String | Unique |
| password_hash | String | bcrypt hash |
| name | String | public name |
| country | String | optional |
| joined_at | Number | epoch timestamp |
| role | String | `user` / `moderator` / `admin` |
| status | String | active, banned |
| avatar_url | String | optional |

---

# 4. Comment System (Future)

Comments will be linked to articles.

Table Name: **F1Comments**  
Partition Key: `article_id`  
Sort Key: `comment_id`  

| Field | Type | Notes |
|-------|------|-------|
| article_id | String | maps to F1Articles.article_id |
| comment_id | String | UUID |
| user_id | String | commenter |
| content | String | text content |
| created_at | Number | epoch |
| parent_comment_id | String | for threaded replies (optional) |
| likes | Number | optional |
| status | String | visible/hidden/deleted |

---

# 5. API Endpoints (Phase 2/3 Only)

### **1. Register**
`POST /v1/auth/register`  
Payload:
```json
{
  "email": "...",
  "name": "...",
  "password": "..."
}
```

### **2. Login**
`POST /v1/auth/login`

### **3. Post Comment**
`POST /v1/comments/{article_id}`  
Requires JWT.

### **4. Get Comments for Article**
`GET /v1/comments/{article_id}`  
Public.

### **5. Delete/Moderate Comment**
`DELETE /v1/comments/{article_id}/{comment_id}`  
Moderator/Admin only.

---

# 6. Security Architecture

- All passwords hashed (bcrypt)
- JWT expires every 7 days
- Refresh-token approach (optional)
- Rate limiting on comment POST
- Anti-spam protection:
  - Cooldown between comments
  - Maximum comment length
  - Block abusive IPs
- Moderation tools as separate admin interface (later)

---

# 7. Frontend Requirements (Future Only)

### Login/Register UI  
- Modal-based or separate page
- Form: name, email, password

### Comment Box UI  
Only visible if user is logged in.

### Comment Thread Display  
- Show list of comments under article
- Show user avatar/name/time

Everything is optional until later phases.

---

# 8. What You Need to Prepare (Later)

You will create later (NOT NOW):

- DynamoDB table *F1Users*
- DynamoDB table *F1Comments*
- Lambda functions:
  - registerUser
  - loginUser
  - postComment
  - getComments
  - deleteComment
- IAM roles:
  - Allow DynamoDB access
  - Allow JWT secret retrieval from Secrets Manager
- Optional:  
  Add CloudFront Functions for rate limiting.

---

# 9. How This Fits Into Phase 1 (Right Now)

Phase 1 (current goal) only needs:

✔ Article API  
✔ Frontend rendering  
✔ Pagination  
✔ Category pages  
✔ Tag pages  
✘ No commenting  
✘ No login  
✘ No user table  

BUT this document ensures the architecture **won’t need redesign** later.

---

# 10. Related Documents

- `01-overview.md`
- `02-dynamodb-schema.md`
- `03-api-gateway-design.md`
- `04-lambda-read-api.md`
- `05-frontend-integration.md`
- `07-site-structure.md`

---