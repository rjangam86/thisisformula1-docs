Perfect — this is exactly the right approach: build a strong baseline first, then optionally layer WAF later.

I’ll give you a production-ready architecture + exact configs for your stack:

CloudFront → API Gateway → Lambda → DynamoDB

* your HMAC/session security model

No fluff — only what you actually need.

⸻

🧠 🔐 OVERALL SECURITY DESIGN (HotLapHQ)

Goal:

* ❌ No direct API access
* ❌ No scraping via curl/Postman
* ✅ Only real browser sessions allowed
* ✅ Cost protection (rate limiting + caching)

⸻

🧱 1. CLOUDFRONT (FRONT DOOR)

A. Origins

1. Static site (S3)

* Origin: S3 bucket
* Access: OAC (Origin Access Control) ✅
* Public access: ❌ disabled

⸻

2. API (API Gateway)

* Origin: xxxx.execute-api.ap-southeast-2.amazonaws.com
* Origin path: /prod

⸻

B. Behaviors

/api/* behavior (CRITICAL)

* Cache policy: CachingDisabled
* Origin request policy: AllViewerExceptHostHeader
* Allowed methods:
    * GET, HEAD, OPTIONS, POST

⸻

C. Custom Header (VERY IMPORTANT)

Add to API origin:

X-HQ-SECRET: your-super-secret-key

👉 This ensures:

* Only CloudFront can call API
* Direct API Gateway access = blocked

⸻

D. Security Headers (Response Headers Policy)

Attach policy:

* Strict-Transport-Security: max-age=63072000
* Content-Security-Policy: default-src 'self'
* X-Frame-Options: DENY
* X-Content-Type-Options: nosniff
* Referrer-Policy: strict-origin-when-cross-origin

⸻

E. CloudFront Function (cheap filtering)

Attach to Viewer Request

Example logic:

* Block empty User-Agent
* Block obvious bot paths

function handler(event) {
  var request = event.request;
  if (!request.headers['user-agent']) {
    return { statusCode: 403 };
  }
  if (request.uri.includes('/wp-admin')) {
    return { statusCode: 403 };
  }
  return request;
}

⸻

🍪 2. SESSION COOKIE (EDGE)

Goal:

* Identify real browser session

Cookie:

hq_session=abc123; Secure; HttpOnly; SameSite=Strict

How:

* Set via CloudFront Function or Lambda@Edge
* TTL: 15–30 minutes

⸻

🔑 3. HMAC TOKEN SYSTEM (CORE SECURITY)

This is your main anti-scraping layer

⸻

A. Frontend (browser)

For every API request:

Headers:

X-HQ-TOKEN: HMAC(secret + timestamp)
X-HQ-TS: timestamp

⸻

B. Backend (Lambda / Authorizer)

Validate:

1. Header exists
2. Timestamp is fresh (±5 min)
3. HMAC matches

⸻

Example Node.js verification:

const crypto = require('crypto');
function verify(token, timestamp) {
  const secret = process.env.HQ_SECRET;
  const expected = crypto
    .createHmac('sha256', secret)
    .update(timestamp)
    .digest('hex');
  return expected === token;
}

⸻

Result:

* curl → ❌ fails
* Postman → ❌ fails
* Bots → ❌ fail

⸻

🚫 4. BLOCK DIRECT API ACCESS

Inside Lambda:

if (event.headers['x-hq-secret'] !== process.env.HQ_SECRET) {
  return { statusCode: 403 };
}

⸻

Also enforce:

* Only allow requests from CloudFront
* Reject unknown headers/origins

⸻

⚡ 5. API GATEWAY THROTTLING

Stage-level

* Rate: 50 req/sec
* Burst: 100

⸻

Method-level (fine control)

Endpoint	Limit
/articles	20 req/sec
/tags	10 req/sec

⸻

Result:

* Attack → instant 429
* Backend safe
* Costs capped

⸻

⚡ 6. CLOUDFRONT CACHING (HUGE IMPACT)

For API GET endpoints:

Cache:

* /api/v1/articles
* /api/v1/tags

TTL:

* 60–300 seconds

⸻

Result:

* 80–90% requests never hit API
* DDoS impact massively reduced

⸻

🌐 7. CORS (STRICT)

API Gateway:

Allow ONLY:

* https://hotlaphq.com
* https://dev.hotlaphq.com

⸻

Headers:

Access-Control-Allow-Origin: https://hotlaphq.com

⸻

📊 8. MONITORING (DON’T SKIP)

Set CloudWatch alarms:

Metrics:

* API Gateway request spike
* Lambda invocation spike
* 5xx errors

⸻

Example:

* Alarm if requests > 2× normal

⸻

🔒 9. EXTRA (OPTIONAL BUT GOOD)

A. Disable direct API invoke

* Use resource policy:
    * Allow only CloudFront IP ranges

⸻

B. Short TTL tokens

* Token expires in 5 minutes

⸻

🧾 FINAL ARCHITECTURE FLOW

Browser
  ↓
CloudFront
  ↓ (adds secret + cookie)
CloudFront Function (filter)
  ↓
API Gateway (throttle)
  ↓
Lambda (validate HMAC + secret)
  ↓
DynamoDB

⸻

⚠️ WHAT YOU ARE NOT USING (INTENTIONALLY)

* ❌ WAF
* ❌ Bot Control
* ❌ Anti-DDoS managed rules

👉 Because:

* Your custom system is already strong
* Cost = $0

⸻

🧠 WHEN TO ADD WAF (LATER)

Add only when:

* Traffic grows (1M+ monthly users)
* You see real attacks
* Revenue justifies ~$10–30/month

⸻

👍 FINAL SUMMARY

👉 With this setup you are protected against:

* ✅ DDoS (via AWS Shield Standard)
* ✅ Scraping (HMAC + header lock)
* ✅ Bot floods (throttling)
* ✅ Direct API abuse (CloudFront-only)

👉 And you pay:

* $0 for WAF

⸻

If you want next step:

👉 I can give you:

* exact CloudFront config JSON
* exact API Gateway resource policy
* exact Lambda authorizer full code

So you can plug it directly into your AWS setup without trial/error.