Perfect. Now we move from API contract → Security Enforcement Contract.

You are correct — the previous MD froze endpoints but did NOT freeze:
	•	CloudFront cookie format
	•	HMAC header format
	•	Lambda Authorizer validation logic
	•	Required server-side validation checks
	•	Client-side token handling
	•	Rate limiting expectations
	•	Rejection rules

We now freeze Security Contract v1 (MVP Production).

Below is your revised implementation document.

⸻

📄 API_SECURITY_IMPLEMENTATION_V1.md

Status: Design Frozen – Ready for Implementation

⸻

1️⃣ Security Architecture Overview

Production Flow:

Browser
   ↓
CloudFront
   ↓ (inject session cookie)
Lambda@Edge (optional lightweight)
   ↓
API Gateway
   ↓ (Lambda Authorizer validates HMAC)
Business Lambda
   ↓
DynamoDB

Dev Environment:
	•	Rate limit only
	•	No HMAC
	•	No cookie enforcement

Production:
	•	Cookie required
	•	HMAC required
	•	WAF enabled
	•	Rate limit enabled

⸻

2️⃣ CloudFront Session Cookie (Production Only)

Cookie Name

cf_session

Format

cf_session = base64(hmacSignature.timestamp.nonce)

Where:
	•	timestamp = epoch seconds
	•	nonce = random UUID
	•	hmacSignature = HMAC_SHA256(secret, timestamp + nonce)

⸻

TTL

30 seconds

Short lived.

⸻

Cookie Flags

HttpOnly
Secure
SameSite=Lax
Path=/api/


⸻

What CloudFront Does

On first HTML load:
	1.	Generate timestamp
	2.	Generate nonce
	3.	Generate HMAC
	4.	Inject cookie into response

Postman will NOT receive this because:
	•	Cookie only set when loading HTML page
	•	API-only calls do not trigger cookie mint

⸻

3️⃣ Required Request Headers (Production)

All API requests must include:

Cookie: cf_session=...
X-Api-Timestamp: <epoch>
X-Api-Signature: <hmac>


⸻

HMAC Header Format

X-Api-Signature = HMAC_SHA256(secret, method + path + timestamp)

Example:

GET/api/v1/teams1700000000


⸻

4️⃣ Lambda Authorizer – Required Validation Steps

Lambda Authorizer MUST:
	1.	Extract cf_session cookie
	2.	Validate signature
	3.	Validate timestamp < 30 seconds
	4.	Validate nonce not replayed (optional Redis/Dynamo short TTL store)
	5.	Validate X-Api-Timestamp
	6.	Validate X-Api-Signature
	7.	Recompute HMAC
	8.	Compare constant-time
	9.	Reject if mismatch

If valid → allow execution
If invalid → 401 Unauthorized

⸻

5️⃣ Business Lambda Required Security Checks

Every Lambda must:

5.1 Query Parameter Validation
	•	limit max 50
	•	cursor max length 200
	•	reject unknown parameters

Example:

Allowed:

limit
cursor
season
teamId
type
tag

Reject anything else.

⸻

5.2 Hard Caps

if (limit > 50) limit = 50;


⸻

5.3 Reject Wildcard Origin

CORS must only allow:

https://thisisformula1.net
https://www.thisisformula1.net

No *

⸻

5.4 Reject Invalid Path

Only allow defined routes.
Anything else → 404 JSON error.

⸻

5.5 Rate Limiting

WAF Rule:
	•	100 requests / 5 minutes per IP
	•	Block if exceeded

⸻

6️⃣ Health Endpoint Exception

/api/v1/health:
	•	No DB access
	•	No cookie required (optional)
	•	Lightweight

⸻

7️⃣ Client-Side Implementation Example

Browser JS must:
	1.	Read cookie automatically (HttpOnly so not accessible via JS)
	2.	Add headers per request

Example:

async function apiFetch(path) {
  const timestamp = Math.floor(Date.now() / 1000);

  const signature = await generateHmac(
    "GET" + path + timestamp
  );

  return fetch(path, {
    method: "GET",
    credentials: "include",
    headers: {
      "X-Api-Timestamp": timestamp,
      "X-Api-Signature": signature
    }
  });
}

Note:
	•	credentials: “include” required
	•	Browser automatically sends cf_session cookie

Postman will fail because:
	•	No valid cookie
	•	No valid HMAC
	•	No timestamp alignment

⸻

8️⃣ Secret Storage

HMAC secret must:
	•	Be stored in AWS Secrets Manager
	•	Not hardcoded
	•	Rotatable

Environment Variable:

HMAC_SECRET


⸻

9️⃣ Error Response Standard

401:

{
  "error": "Unauthorized"
}

400:

{
  "error": "Invalid request"
}

429:

{
  "error": "Rate limit exceeded"
}


⸻

🔟 Production Checklist

Before launch:
	•	WAF enabled
	•	Rate limit rule active
	•	Cookie injection active
	•	Lambda Authorizer deployed
	•	HMAC secret stored in Secrets Manager
	•	CORS locked
	•	Limit caps enforced

⸻

🔒 Summary

Layer	Protection
CloudFront	Session cookie
API Gateway	Lambda Authorizer
Lambda	Input validation
WAF	Rate limiting
Secrets Manager	HMAC secret


⸻

Status

Security Contract v1 – Frozen
Ready for Implementation