Good. Monitoring for the next 24 hours is the correct move now.

And YES — if you stabilize around:

~1000–1300 requests/hour

you should comfortably survive the remaining 6 days.

⸻

Now about:

AWS WAF + DDoS protection

Good news:
you ALREADY have:

AWS Shield Standard

automatically included for free.

That protects against:

* volumetric DDoS
* SYN floods
* UDP floods
* network-layer attacks

You do NOT need to enable anything manually for basic Shield Standard.

⸻

What you CAN additionally enable

AWS WAF

This is separate.

WAF protects:

* bots
* abusive requests
* scraping
* rate abuse
* malicious patterns

This is the one useful for HotLapHQ later.

⸻

IMPORTANT

AWS WAF is NOT fully free.

There are charges:

* Web ACL monthly fee
* per-rule fee
* per-request inspection fee

BUT:
for your current scale,
it would likely still be small.

Probably:

~$5–15/month

depending on traffic/rules.

⸻

My recommendation RIGHT NOW

DO NOT enable WAF yet.

Because:

* your issue was mostly uncached assets
* not a real attack
* traffic already collapsing
* bots only ~12%
* infrastructure healthy

So:

no urgent need

right now.

⸻

What you SHOULD do later

After traffic stabilizes:

* enable AWS WAF
* add simple rate limiting
* maybe bot control later

⸻

BEST future starter setup

Eventually:

Rule 1

Rate limit:

200 requests / 5 min per IP

⸻

Rule 2

Block obvious bad bots:

* empty user agents
* suspicious headers

⸻

Rule 3

Geo/rate monitoring only initially

⸻

VERY IMPORTANT

Do NOT over-tighten WAF early.

Because your site:

* SPA
* many API calls
* many assets
* HMAC session architecture

Aggressive WAF can accidentally:

* block real users
* break Safari
* break CloudFront session bootstrap

So later we do:

monitor mode first

not hard blocking immediately.

⸻

Current status summary

Area	Status
CloudFront caching	FIXED
Fake 500 errors	FIXED
Session recovery	FIXED
Data transfer	VERY healthy
Requests	improving
DDoS protection	already included
WAF	optional later

You are in a much safer position now than yesterday.