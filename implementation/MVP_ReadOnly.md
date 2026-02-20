Document 1 — MVP Public Read-Only API Surface (v1)

Scope
	•	MVP is read-only.
	•	All endpoints are served under the main site domain:
	•	https://<domain>/api/*
	•	No separate API hostname (no api.<domain>).

⸻

Articles & Tags (existing, migrated under /api)

List articles
GET /api/articles?limit={number}&cursor={cursor}
	•	Purpose: homepage feed, pagination.
	•	Query params:
	•	limit (required/optional per implementation; enforce max)
	•	cursor (opaque pagination token)
	•	Notes:
	•	Reject unknown query params (cost + cache protection).
	•	cursor should remain opaque (do not expose internal keys).

Article detail
GET /api/articles/{slug_id}
	•	Purpose: article detail page.
	•	Path params:
	•	slug_id (canonical slug + UID string)

Articles by type
GET /api/articles/type/{type}?limit={number}&cursor={cursor}
	•	Purpose: opinion blocks / category pages.
	•	Path params:
	•	type (allow-list accepted values)

Articles by tag
GET /api/articles/tag/{tag}?limit={number}&cursor={cursor}
	•	Purpose: tag pages.
	•	Path params:
	•	tag (normalize to lowercase; input cap)

Tags list
GET /api/tags?limit={number}
	•	Purpose: tags index / UI typeahead (if used).
	•	Query params:
	•	limit (cap)

⸻

Race Context (new MVP endpoints)

Teams
GET /api/teams
	•	Purpose: Teams page (team-centric).
	•	Caching: long (e.g., hours/day) since rarely changes.

Standings — Drivers
GET /api/standings/drivers
	•	Purpose: Drivers standings widget/page.
	•	Source: standings snapshot table/materialized view (read-only).
	•	Caching: moderate (minutes/hours depending on update cadence).

Standings — Constructors
GET /api/standings/constructors
	•	Purpose: Constructors standings widget/page.
	•	Source: standings snapshot table/materialized view.

Calendar — Full season sessions
GET /api/calendar
	•	Purpose: calendar page (sessions list).
	•	Source: calendar sessions table (read-only).
	•	Caching: moderate to long.

Calendar — Next session (countdown widget)
GET /api/calendar/next-session
	•	Purpose: next FP/Quali/Race countdown widget.
	•	Source: query by time (GSI) or precomputed “next session”.
	•	Caching: short to moderate (minutes).

⸻

Optional (operational)

GET /api/health (optional)
	•	Purpose: internal checks / monitoring only.
	•	Can be WAF-restricted.

⸻

General API rules (MVP)
	•	Methods allowed: GET, HEAD, OPTIONS only.
	•	No public write APIs.
	•	Input caps:
	•	limit max (e.g., 50)
	•	cursor max length
	•	tag/type allow-lists or strict validation
	•	reject unknown query params
	•	Response format: JSON only (application/json)
	•	Errors: consistent JSON envelope, no stack traces.

⸻

Document 2 — MVP Security Architecture Reference (Edge-Gated Read-Only APIs)

Objective (non-negotiable)
	1.	Prevent direct API Gateway access (no bypassing CloudFront).
	2.	Prevent third-party websites from consuming your APIs to clone your site (cross-site API reuse).
	3.	Reduce abuse risk (scraping, brute force, cost blowups) via WAF + caching + validation.
	4.	Keep MVP constraints: no user accounts, no login UI, no public write APIs.

⸻

High-level architecture (verbal diagram)

Static site
User → CloudFront → S3 (private bucket via OAC)

APIs
User → CloudFront → WAF → API Gateway (Regional) → Lambda Authorizer → Read-only Lambda → DynamoDB

All API paths are under:
	•	https://<domain>/api/*

There is no api.<domain> hostname.

⸻

Core security concept

CloudFront cannot inherently tell “browser vs Postman.”
So identity is established through signed state that only a same-site browser session can possess, and that CloudFront validates before allowing API calls.

That signed state is a short-lived, signed session cookie issued at the edge.

⸻

Components and responsibilities

1) CloudFront (single distribution)
	•	Serves both:
	•	static site (/*) from S3
	•	APIs (/api/*) from API Gateway
	•	Enforces HTTPS.
	•	Hosts the edge logic (CloudFront Functions and/or Lambda@Edge).

Behaviors
	•	Default * → S3 origin (GET/HEAD)
	•	/api/* → API Gateway origin (GET/HEAD/OPTIONS)

⸻

2) WAF (attached to CloudFront)
Purpose: abuse prevention & cost protection, not identity.

Recommended baseline:
	•	AWS Managed Common Rules
	•	Known bad inputs
	•	IP reputation list
	•	Rate limit for /api/* (example: 60 req / 5 minutes / IP; tune after launch)

WAF blocks:
	•	high-rate scraping attempts
	•	brute force / token guessing at scale
	•	common malicious payload patterns

⸻

3) Edge session cookie (bootstrap)
On first page load:
	•	User requests: GET / (or any HTML route)
	•	CloudFront edge logic issues a cookie:

Example properties:
	•	__site_session=<signed-value>
	•	HttpOnly; Secure; SameSite=Lax; Max-Age=<minutes>

Key properties:
	•	Signed (HMAC) with a secret only edge+authorizer know
	•	HttpOnly (not readable by JS)
	•	SameSite=Lax (blocks cross-site contexts)

Why this matters:
	•	A browser automatically stores and sends cookies back to the same domain.
	•	Another website cannot cause a browser to send this cookie cross-site.
	•	Postman/curl won’t have this cookie unless they explicitly simulate the bootstrap step.

⸻

4) Edge-gated API token injection (per-request)
When a request hits /api/*:

Edge logic performs:
	1.	Check for __site_session cookie
	2.	Validate cookie signature + expiry
	3.	If valid:
	•	Generate a short-lived API token (HMAC signed), scoped to:
	•	path (and optionally method)
	•	expiry (e.g., 30–60 seconds)
	•	Inject token into the origin request as a header:
	•	x-api-token: <signed-token>
	4.	If invalid or missing cookie:
	•	Reject at edge (403) or forward without token (authorizer will reject)

Important: the API token is created inside CloudFront, not by JavaScript.

⸻

5) API Gateway (Regional) + “no bypass” controls
	•	API Gateway is the origin behind CloudFront.
	•	Direct access must be blocked:
	•	No public custom domain for API Gateway.
	•	Apply an API Gateway resource policy and/or authorizer gating such that:
	•	requests without valid authorization fail
	•	Only CloudFront-served requests (carrying valid x-api-token) succeed.

⸻

6) Lambda Authorizer (hard gate)
Runs before your read-only Lambda handler.

Validations:
	•	x-api-token exists
	•	HMAC signature is valid (shared secret)
	•	token not expired
	•	token scope matches request path/method (recommended)

If any validation fails → 403 (fast reject)

⸻

Why cloned websites fail (your main requirement)

A third-party site (different domain) tries:

fetch("https://<domain>/api/articles")

In a browser:
	•	No __site_session cookie is sent (SameSite + cross-site rules)
	•	Edge does not inject x-api-token
	•	Authorizer rejects
	•	Result: 403

This prevents them from consuming your API from their website to replicate yours.

⸻

Why Postman typically fails

If Postman calls:

GET https://<domain>/api/articles
	•	No signed session cookie → edge won’t inject token → authorizer rejects → 403

If Postman tries to spoof:
	•	Cannot mint a valid signed cookie (no secret) → fails validation

Note: a determined actor could simulate a full browser session, but:
	•	WAF rate limits make high-scale scraping expensive
	•	tokens are short-lived and path-scoped
	•	you still block cross-site reuse (the core “clone my site” threat)

⸻

Caching and cost protection (required)
	•	CloudFront caching on /api/* reduces origin cost and absorbs spikes.
	•	Cache key allow-lists query params (e.g., limit, cursor, tag, type).
	•	Reject unknown query params to prevent cache-busting abuse.
	•	Browser-side caching is helpful but not a substitute for CloudFront caching.

⸻

MVP security guarantees (what this architecture delivers)

✅ No direct public access to S3 (OAC)
✅ No direct usable access to API Gateway (authorizer-gated)
✅ Third-party sites cannot call your APIs from their own domain (cross-site blocked)
✅ Anonymous Postman/curl calls fail (no session cookie / no token)
✅ Abuse is controlled via WAF rate limiting + strict validation + caching

Out of MVP scope:
	•	user accounts / login
	•	CAPTCHA flows
	•	fully preventing any determined, low-volume manual scraping forever (not realistically achievable for public content)

⸻

Tickets (architecture reference)
	•	ARCH-003: WAF baseline + rate limits
	•	ARCH-004: Session cookie issuance at edge
	•	ARCH-005: API token injection at edge
	•	ARCH-006: Lambda Authorizer token validation
	•	ARCH-007: Disable direct API Gateway bypass
	•	ARCH-010+: Teams/Calendar/Standings tables + endpoints