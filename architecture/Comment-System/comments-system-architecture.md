HotLapHQ Comment System Architecture (Draft v1)

Overview

This document defines the proposed architecture for the HotLapHQ comment system.

The design follows the same principles as the existing HotLapHQ platform:

* Serverless-first
* Low operational overhead
* Low fixed monthly costs
* High scalability
* Strong anti-abuse protections
* Decoupled static/dynamic architecture

The system intentionally keeps:

* Articles fully static and CDN-cached
* Comments fully dynamic and API-driven

⸻

1. Architectural Goals

Primary Goals

* Add authenticated comments to articles
* Support nested replies
* Protect against bots/spam/fake accounts
* Maintain ultra-low infrastructure costs
* Preserve existing S3 + CloudFront architecture
* Scale automatically during viral traffic spikes
* Avoid continuous-running servers
* Maintain strong ownership/control over data

⸻

2. Existing HotLapHQ Architecture

Current architecture:

Browser
  ↓
CloudFront
  ↓
S3 Static Website
  ↓
Client-side JavaScript
  ↓
API Gateway
  ↓
Lambda
  ↓
DynamoDB

Articles:

* Stored in S3
* Metadata in DynamoDB
* Rendered client-side
* Cached aggressively by CloudFront

This architecture remains unchanged.

⸻

3. Comment System High-Level Design

Comment Architecture

User Browser
  ↓
CloudFront
  ↓
Static Article Page
  ↓
Comment Widget JavaScript
  ↓
API Gateway (/api/v1/comments/*)
  ↓
Lambda Functions
  ↓
DynamoDB

The comment system is completely separated from article storage.

⸻

4. Core AWS Services

AWS Services Used

Service	Purpose
Amazon S3	Static article storage
CloudFront	CDN + caching
API Gateway	Comment APIs
Lambda	Business logic
DynamoDB	Comment/user storage
Cognito	Authentication
CloudWatch	Logging/monitoring
IAM	Access control
Route53	DNS

⸻

5. Services NOT Used Initially

Not Recommended Initially

Service	Reason
ECS/Fargate	Operational overhead unnecessary
EC2	Continuous server cost
RDS	Not needed for comment workload
ElastiCache	Premature optimization
OpenSearch	Not needed initially
AWS WAF	Optional later due to fixed monthly cost

⸻

6. Cloudflare Usage

Cloudflare Strategy

Cloudflare will NOT replace CloudFront.

Recommended usage:

Cloudflare Feature	Usage
DNS	Yes
Turnstile CAPTCHA	Yes
Proxy/CDN Mode	No initially
Cloudflare Full Reverse Proxy	No initially

Reason:

Avoid:

* caching conflicts
* cookie/session complications
* debugging complexity
* origin/header problems

CloudFront remains the primary CDN.

⸻

7. Authentication Architecture

Authentication Provider

Use Amazon Cognito User Pools.

Login Methods

Enabled

* Google Login
* Apple Login
* Email OTP / Magic Link

Disabled Initially

* Username/password accounts

Reason:

* Lower friction
* Better security
* Reduced password-reset overhead
* Reduced credential attack surface

⸻

8. Comment Security Stack

Security Layers

Layer 1 — Cognito Authentication

Only authenticated users can post comments.

Anonymous reading remains public.

⸻

Layer 2 — Cloudflare Turnstile

Frontend obtains Turnstile token.

Token submitted with comment payload.

Lambda verifies token before write.

⸻

Layer 3 — API Gateway Throttling

Throttle comment APIs.

Example:

Endpoint	Rate
POST /comments	Low
POST /reply	Low
POST /report	Very low

⸻

Layer 4 — Lambda Abuse Validation

Lambda validates:

* account age
* cooldown timers
* spam patterns
* URL count
* duplicate comments
* excessive posting
* banned users
* shadow-banned users

⸻

Layer 5 — Moderation Layer

Comments may be:

visible
pending
hidden
shadow_hidden
deleted

⸻

9. DynamoDB Design

Main Table

Table Name

HotLapHQComments

⸻

Table Keys

Partition Key

PK = ARTICLE#<articleId>

Sort Key

SK = COMMENT#<timestamp>#<commentId>

⸻

10. Comment Item Structure

Top-Level Comment

{
  "PK": "ARTICLE#verstappen-monaco-win",
  "SK": "COMMENT#2026-05-25T12:00:00Z#cmt_001",
  "commentId": "cmt_001",
  "articleId": "verstappen-monaco-win",
  "parentCommentId": null,
  "userId": "usr_123",
  "displayName": "Rahul",
  "content": "Amazing race pace today.",
  "status": "visible",
  "createdAt": "2026-05-25T12:00:00Z",
  "updatedAt": null,
  "replyCount": 0,
  "likeCount": 0,
  "reportCount": 0,
  "accountAgeDays": 10,
  "ipHash": "sha256",
  "deviceHash": "sha256"
}

⸻

11. Nested Replies

Reply Structure

Replies are stored in same table.

{
  "parentCommentId": "cmt_001"
}

Frontend builds nested tree client-side.

Advantages:

* single query
* cheap reads
* no recursive DB operations
* scalable

⸻

12. Comment Read Flow

Read Flow

Browser
  ↓
GET /api/v1/comments?articleId=xxx
  ↓
API Gateway
  ↓
Lambda
  ↓
DynamoDB Query
  ↓
Return Flat Comment List
  ↓
Frontend Builds Tree

⸻

13. Comment Write Flow

Write Flow

User submits comment
  ↓
Turnstile token generated
  ↓
POST /api/v1/comments
  ↓
API Gateway
  ↓
Lambda
  ↓
Verify Cognito JWT
  ↓
Verify Turnstile token
  ↓
Validate cooldown/rate limits
  ↓
Validate moderation rules
  ↓
Write to DynamoDB

⸻

14. Frontend Rendering

Frontend Responsibilities

Frontend comment widget handles:

* fetch comments
* render nested tree
* submit comments
* submit replies
* optimistic rendering
* lazy loading replies
* report comment
* moderation state rendering

⸻

15. CloudFront Caching Strategy

Comment Reads

Allow short-term caching:

15–60 seconds

Benefits:

* viral traffic protection
* lower DynamoDB reads
* lower Lambda invocations

⸻

Important

After posting comment:

Frontend should:

* optimistically inject comment
    OR
* refetch with cache-busting query param

Otherwise user may not immediately see comment.

⸻

16. Anti-Spam Rules

Initial Rules

User Cooldowns

Rule	Value
Same article comment cooldown	2 minutes
Global comment cooldown	configurable
Reply cooldown	configurable

⸻

New Account Restrictions

Restriction	Enabled
Link posting blocked	Yes
Higher moderation sensitivity	Yes
Stricter cooldowns	Yes

⸻

Spam Detection Checks

Lambda checks:

* repeated content
* excessive links
* copied spam
* unicode abuse
* suspicious patterns

⸻

17. Moderation Architecture

Moderation Actions

Admins can:

* approve
* hide
* delete
* restore
* ban user
* shadow-ban user

⸻

Shadow Ban

Shadow-hidden comments:

* visible to spammer
* invisible to public

Useful against persistent spam accounts.

⸻

18. User Reputation System (Future)

Future optional features:

* trusted users
* reputation score
* auto-approved users
* verified users
* spam scoring
* report thresholds

⸻

19. Admin Moderation Panel

Recommended Later

Separate admin UI:

/admin/comments

Functions:

* review pending comments
* review reports
* ban users
* search comments
* moderation logs

⸻

20. Monitoring & Logging

CloudWatch Logging

Log:

* failed Turnstile verification
* spam rejections
* excessive rate limit hits
* banned user attempts
* API failures

⸻

21. Legal & Compliance Updates

Update:

* Privacy Policy
* Cookie Policy
* Terms of Use
* Community Guidelines

Mention:

* public comments
* moderation rights
* account handling
* abuse prevention
* anti-spam logging

⸻

22. Cost Strategy

Primary Cost Philosophy

Maintain:

* serverless
* on-demand billing
* no continuously running infrastructure

⸻

Expected Cost Drivers

Primary future costs:

* moderation time
* viral traffic
* storage growth

Not:

* Lambda
* DynamoDB initially

⸻

23. Recommended Initial MVP Scope

Phase 1

Implement:

* Cognito login
* Turnstile
* top-level comments
* nested replies
* moderation states
* basic admin moderation
* DynamoDB storage
* API Gateway throttling

⸻

Phase 2

Add:

* likes
* reports
* reputation system
* trusted users
* notifications
* moderation analytics

⸻

24. Final Recommended Stack

Final Architecture

CloudFront
  ↓
Static Article Site (S3)
  ↓
Comment Widget
  ↓
API Gateway
  ↓
Lambda
  ↓
DynamoDB
Authentication:
Cognito
Bot Protection:
Cloudflare Turnstile
Monitoring:
CloudWatch

⸻

25. Key Architectural Decisions

Final Decisions

Area	Decision
Article storage	S3
Comment storage	DynamoDB
Authentication	Cognito
CAPTCHA	Cloudflare Turnstile
CDN	CloudFront
Cloudflare proxy mode	Disabled initially
Backend compute	Lambda
Database	DynamoDB
Bot protection	Turnstile + throttling
Moderation	Internal
Third-party comments	Avoid
ECS/Fargate	Not initially