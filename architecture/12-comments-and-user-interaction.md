# 12 — Comments & User Interaction  
_ThisIsFormulaOne Platform — User Feedback Layer_

This document defines how article comments, user identities, basic interaction endpoints, and moderation workflows will be implemented and integrated into the existing read-only architecture.  
Comments & interactions are **Phase 2** features but must be architected now so the system remains extensible.

---

## 1. Goals & Philosophy

The comments system must be:

- **Lightweight at launch** (optional to enable)
- **Scalable** (millions of comments possible)
- **Secure** (spam-resistant, rate-limited)
- **Modular** (future integration with user accounts)
- **Non-blocking** (frontend should load even without comments)
- **Compatible with static site caching** via CloudFront

---

## 2. High-Level Architecture

| Component | Responsibility |
|----------|----------------|
| **Frontend (JS)** | Fetch comments, render thread, submit new comment |
| **API Gateway (REST)** | Exposes `/v1/comments/*` endpoints |
| **Lambda – Comment Handler** | Validates, sanitizes, stores comments |
| **DynamoDB – Comments Table** | Stores comments, replies, moderation state |
| **Lambda – Moderation Worker (future)** | AI-based or rule-based moderation |
| **CloudFront** | Caches article pages; comment API is not cached |
| **Identity Layer** | Guest mode initially, user accounts later |

---

## 3. DynamoDB Schema for Comments

**Table Name:** `F1ArticleComments`

### Primary Keys

| Field | Type | Description |
|-------|------|-------------|
| `comment_id` | String (PK) | UUID for each comment |
| `article_id` | String (GSI1 PK) | Maps comments to articles |
| `created_at` | Number (GSI1 SK) | Timestamp for sorting |

### Core Attributes

| Field | Type | Description |
|-------|------|-------------|
| `user_id` | String | Optional; null in guest mode |
| `username` | String | Display name |
| `content` | String | Markdown-safe sanitized text |
| `status` | String | `VISIBLE`, `PENDING`, `HIDDEN`, `DELETED` |
| `parent_id` | String | For threaded replies |
| `ip_hash` | String | Rate-limiting + anti-abuse |
| `likes` | Number | Upvotes |
| `flags` | Number | Abuse flag count |

---

## 4. Endpoints for Comments API

### A. List Comments  
**GET** `/v1/comments/{article_id}`

Returns comments in ascending chronological order, threaded.

Response:

```json
{
  "data": [
    {
      "comment_id": "123",
      "username": "Carlos",
      "content": "Ótima matéria!",
      "created_at": 1763511001,
      "likes": 3,
      "replies": []
    }
  ]
}
```

---

### B. Create Comment  
**POST** `/v1/comments`

Payload:

```json
{
  "article_id": "leclerc-signs-extension",
  "username": "Guest123",
  "content": "Ferrari finally doing something right!"
}
```

Validation steps inside Lambda:

- Sanitize HTML  
- Limit length  
- Rate limit via IP hash  
- Set status → `VISIBLE` or `PENDING` depending on moderation setting  

---

### C. Like a Comment  
**POST** `/v1/comments/{comment_id}/like`

Increments the like counter.  
Anti-abuse: 1 like per IP hash.

---

### D. Flag a Comment  
**POST** `/v1/comments/{comment_id}/flag`

If `flags > threshold`, auto-hide comment.

---

## 5. Moderation Modes

### Mode A — **Auto-Publish (default)**  
- Comments appear immediately  
- Auto-hide if flagged too often

### Mode B — **Pre-Moderation**  
- All comments go to `PENDING`  
- Admin must approve via Admin Panel  

### Mode C — **AI Moderation (future)**  
- A Lambda worker analyzes text  
- Uses filters for slurs, spam, repeated content  

---

## 6. Guest Mode (Phase 1)

Guest users provide:

- Display name  
- (Optional) email  
- No password  
- IP-hash stored for rate limiting  

This avoids needing full user registration initially.

---

## 7. Future: Registered Users (Phase 3)

Once you implement user accounts:

- Login required for comments  
- Users can view comment history  
- Profile picture support  
- Ban system  
- OAuth via Google/Apple  

Comment table already supports `user_id` for seamless upgrade.

---

## 8. Frontend Integration

### A. Load Comments on Article Page
Triggered by JS once article loads:

```js
fetch(`/v1/comments/${articleId}`)
  .then(res => res.json())
  .then(renderComments);
```

---

### B. Comment Submission Form

Simple HTML:

```html
<form id="comment-form">
  <input name="username" placeholder="Your name" required>
  <textarea name="content" required></textarea>
  <button type="submit">Post Comment</button>
</form>
```

Backend responds with:

```json
{ "status": "ok", "comment_id": "123" }
```

---

## 9. Anti-Spam & Abuse Controls

- IP hashing (SHA256)  
- 1 comment per 30 seconds cooldown  
- HTML sanitization  
- Blacklist terms list  
- Flood protection (max X comments per hour)  
- Flag-based hiding  

---

## 10. Data Retention Rules

- Comments older than **2 years** → archived or auto-deleted  
- Deleted comments kept for **7 days** for admin review  
- Flags reset every **90 days**  

---

## 11. AWS Roles Needed

**Lambda → DynamoDB:**  
- `GetItem`  
- `PutItem`  
- `UpdateItem`  
- `Query`

**API Gateway → Lambda:**  
- Standard proxy execution role

**Admin Panel (future):**  
- IAM role restricting to moderation APIs only

---

## 12. Future Extensions

- Email notifications for replies  
- @mention users  
- Image/GIF reactions  
- Comment ranking (hot, newest, top)  

---

## 13. Summary

This file defines a **modular, scalable, secure** comments system that can be enabled later without rewriting the backend.  
It integrates with your existing REST API, DynamoDB structure, and frontend layout.