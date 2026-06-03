HOTLAPHQ FINAL SECURITY IMPLEMENTATION ORDER

Implement EXACTLY in this order.

Do NOT jump ahead.

⸻

STEP 1 — SECURE S3

1.1 Block Public Access

For ALL S3 buckets:

* enable ALL Block Public Access settings

⸻

1.2 Configure CloudFront OAC

CloudFront:

* Origins
* S3 origin
* Create Origin Access Control
* Attach OAC

Remove:

* public bucket policy

Result:

* S3 accessible ONLY via CloudFront

⸻

STEP 2 — CREATE API GATEWAY ORIGIN IN CLOUDFRONT

CloudFront:

* Origins
* Add origin

Origin domain:

xxxx.execute-api.ap-southeast-2.amazonaws.com

Origin path:

/prod

Protocol:

HTTPS only

⸻

STEP 3 — ADD CLOUDFRONT SECRET HEADER

CloudFront:

* Origins
* API origin
* Add custom header

Header:

X-HQ-SECRET

Value:

LONG_RANDOM_SECRET_64_CHARS

Save SAME value later in Lambda env variable:

HQ_SECRET

⸻

STEP 4 — CREATE API BEHAVIOR

CloudFront:

* Behaviors
* Create behavior

Path:

/api/*

Settings:

Allowed methods:

GET
HEAD
OPTIONS
POST

Viewer protocol:

Redirect HTTP to HTTPS

Compress:

Enabled

Cache policy:

CachingDisabled

Origin request policy:

AllViewerExceptHostHeader

Origin:

* API Gateway origin

⸻

STEP 5 — CREATE RESPONSE HEADERS POLICY

CloudFront:

* Policies
* Response headers policy
* Create policy

Add:

Strict-Transport-Security:
max-age=63072000; includeSubDomains; preload
Content-Security-Policy:
default-src 'self';
X-Frame-Options:
DENY
X-Content-Type-Options:
nosniff
Referrer-Policy:
strict-origin-when-cross-origin

Attach this policy to:

* default behavior
* /api/*

⸻

STEP 6 — CREATE CLOUDFRONT FUNCTION

CloudFront:

* Functions
* Create function

Attach:

Viewer Request

Code:

function handler(event) {
    var request = event.request;
    if (!request.headers['user-agent']) {
        return {
            statusCode: 403
        };
    }
    var uri = request.uri.toLowerCase();
    var blocked = [
        '/wp-admin',
        '/xmlrpc.php',
        '/phpmyadmin',
        '/.env'
    ];
    for (var i = 0; i < blocked.length; i++) {
        if (uri.includes(blocked[i])) {
            return {
                statusCode: 403
            };
        }
    }
    return request;
}

Publish function.
Associate with distribution.

⸻

STEP 7 — ENABLE API GATEWAY THROTTLING

API Gateway:

* Stages
* prod
* Throttling

Set:

Rate:

50

Burst:

100

⸻

STEP 8 — CREATE SESSION TOKEN LAMBDA

Create Lambda:

session-bootstrap

Add env variable:

HQ_HMAC_SECRET

Value:

* another long random secret

⸻

STEP 9 — ADD TOKEN GENERATION CODE

Inside:

session-bootstrap

Use:

import crypto from "crypto";
const SECRET = process.env.HQ_HMAC_SECRET;
function base64UrlEncode(input) {
  return Buffer.from(input)
    .toString("base64")
    .replace(/\+/g, "-")
    .replace(/\//g, "_")
    .replace(/=+$/, "");
}
export const handler = async () => {
  const now = Math.floor(Date.now() / 1000);
  const payload = {
    iat: now,
    exp: now + 300,
    dom: "hotlaphq.com",
    aud: "hotlaphq-api"
  };
  const encoded = base64UrlEncode(
    JSON.stringify(payload)
  );
  const signature = crypto
    .createHmac("sha256", SECRET)
    .update(encoded)
    .digest("hex");
  const token = `${encoded}.${signature}`;
  return {
    statusCode: 200,
    headers: {
      "Set-Cookie":
        `hq_session=${token}; Path=/; Secure; SameSite=Lax; Max-Age=300`
    },
    body: "ok"
  };
};

⸻

STEP 10 — CREATE SESSION ENDPOINT

API Gateway:

* Create endpoint:

/api/session

Connect to:

session-bootstrap

NO authorizer on this endpoint.

Purpose:

* browser gets token

⸻

STEP 11 — UPDATE FRONTEND FETCH LOGIC

Frontend:

* commons.js
* datastore.js
* central API wrapper

Add:

function getCookie(name) {
  const match = document.cookie.match(
    new RegExp("(^|; )" + name + "=([^;]*)")
  );
  return match
    ? decodeURIComponent(match[2])
    : null;
}
async function apiFetch(url, options = {}) {
  const token = getCookie("hq_session");
  const headers = new Headers(
    options.headers || {}
  );
  headers.set("X-HQ-TOKEN", token);
  headers.set("X-HQ-TS", String(Date.now()));
  return fetch(url, {
    ...options,
    headers,
    credentials: "same-origin"
  });
}

Replace ALL:

fetch(...)

calls to API with:

apiFetch(...)

⸻

STEP 12 — CREATE REQUEST AUTHORIZER

API Gateway:

* Authorizers
* Create authorizer

Type:

REQUEST

Identity sources:

method.request.header.X-HQ-TOKEN
method.request.header.X-HQ-TS
method.request.header.Origin

⸻

STEP 13 — CREATE AUTHORIZER LAMBDA

Create Lambda:

hq-authorizer

Add env variable:

HQ_HMAC_SECRET

⸻

STEP 14 — ADD AUTHORIZER CODE

Inside:

hq-authorizer

Add:

import crypto from "crypto";
const SECRET = process.env.HQ_HMAC_SECRET;
const ALLOWED_ORIGINS = [
  "https://hotlaphq.com",
  "https://dev.hotlaphq.com"
];
function deny(methodArn) {
  return {
    principalId: "deny",
    policyDocument: {
      Version: "2012-10-17",
      Statement: [{
        Action: "execute-api:Invoke",
        Effect: "Deny",
        Resource: methodArn
      }]
    }
  };
}
function allow(methodArn) {
  return {
    principalId: "allow",
    policyDocument: {
      Version: "2012-10-17",
      Statement: [{
        Action: "execute-api:Invoke",
        Effect: "Allow",
        Resource: methodArn
      }]
    }
  };
}
function safeEqual(a, b) {
  const aBuf = Buffer.from(a);
  const bBuf = Buffer.from(b);
  if (aBuf.length !== bBuf.length) {
    return false;
  }
  return crypto.timingSafeEqual(aBuf, bBuf);
}
export const handler = async (event) => {
  try {
    const headers = event.headers || {};
    const token =
      headers["x-hq-token"];
    const origin =
      headers["origin"];
    if (!token) {
      return deny(event.methodArn);
    }
    if (!ALLOWED_ORIGINS.includes(origin)) {
      return deny(event.methodArn);
    }
    const parts = token.split(".");
    if (parts.length !== 2) {
      return deny(event.methodArn);
    }
    const encodedPayload = parts[0];
    const signature = parts[1];
    const expected = crypto
      .createHmac("sha256", SECRET)
      .update(encodedPayload)
      .digest("hex");
    if (!safeEqual(signature, expected)) {
      return deny(event.methodArn);
    }
    const payload = JSON.parse(
      Buffer.from(
        encodedPayload,
        "base64"
      ).toString("utf8")
    );
    const now =
      Math.floor(Date.now() / 1000);
    if (now > payload.exp) {
      return deny(event.methodArn);
    }
    return allow(event.methodArn);
  } catch (err) {
    return deny(event.methodArn);
  }
};

⸻

STEP 15 — ATTACH AUTHORIZER

Attach:

hq-authorizer

to ALL:

/api/v1/*

routes.

EXCEPT:

/api/session

⸻

STEP 16 — ADD LAMBDA SECRET VALIDATION

Inside EVERY API Lambda:

Add:

const secret =
  event.headers["x-hq-secret"];
if (
  secret !== process.env.HQ_SECRET
) {
  return {
    statusCode: 403,
    body: "Forbidden"
  };
}

Add env variable:

HQ_SECRET

with SAME value from CloudFront origin header.

⸻

STEP 17 — ENABLE STRICT CORS

API Gateway:

* CORS

Allow origins ONLY:

https://hotlaphq.com
https://dev.hotlaphq.com

Allow headers:

Content-Type
X-HQ-TOKEN
X-HQ-TS

Allow methods:

GET
POST
OPTIONS

Credentials:

Enabled

⸻

STEP 18 — ENABLE API CACHE

CloudFront:

* Behaviors

Create cache behaviors:

/api/v1/articles*
/api/v1/tags*
/api/v1/drivers*
/api/v1/teams*

TTL:

60–300 seconds

⸻

STEP 19 — ENABLE WAF (PRODUCTION ONLY)

Attach WAF ONLY to:

* hotlaphq.com￼

NOT:

* dev.hotlaphq.com

⸻

STEP 20 — ADD WAF RULES

Enable:

AWSManagedRulesCommonRuleSet
AWSManagedRulesKnownBadInputsRuleSet
AWSManagedRulesAmazonIpReputationList

⸻

STEP 21 — ADD WAF RATE RULE

Create rule:

1000 requests
5 minutes
per IP

Action:

BLOCK

⸻

STEP 22 — ENABLE CLOUDWATCH ALARMS

Create alarms for:

* API Gateway 5XX
* Lambda Errors
* Lambda Throttles
* Request spikes
* WAF spikes

⸻

STEP 23 — FIX IAM PERMISSIONS

For EVERY Lambda:

* least privilege only

NEVER:

*

permissions.

⸻

STEP 24 — MOVE SECRETS LATER

Move later into:
Amazon Web Services Secrets Manager

Secrets:

* HQ_SECRET
* HQ_HMAC_SECRET