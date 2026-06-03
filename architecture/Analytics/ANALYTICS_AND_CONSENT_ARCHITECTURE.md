Perfect — this is exactly the right time to formalize it. I’ll give you a clean, production-grade MD document you can keep in your docs repo and follow.

⸻

📄 ANALYTICS_AND_CONSENT_ARCHITECTURE.md

⸻

1. Overview

HotLapHQ analytics and consent system is designed to:
	•	Track user behavior (pages, articles, drivers, teams)
	•	Comply with GDPR / EEA consent requirements
	•	Be scalable for future features (accounts, comments, likes)

⸻

2. Current MVP Scope (READ THIS FIRST)

What we DO collect:
	•	Page visits
	•	Article views
	•	Driver / Team views
	•	Category & tag engagement
	•	Device, country (via GA)

What we DO NOT collect:
	•	No user accounts
	•	No emails
	•	No personal identifiable data (PII)

👉 This is low-risk GDPR profile

⸻

3. Google Analytics Architecture

3.1 Base Setup

All pages include:

<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}

  gtag('consent', 'default', {
    analytics_storage: 'denied',
    ad_storage: 'denied',
    ad_user_data: 'denied',
    ad_personalization: 'denied',
    wait_for_update: 500
  });

  gtag('js', new Date());
</script>

<script async src="https://www.googletagmanager.com/gtag/js?id=G-XXXX"></script>

<script>
  gtag('config', 'G-XXXX', {
    send_page_view: false
  });
</script>


⸻

3.2 Tracking Model

Page Tracking

trackPage(pageType, pagePath, params)

Event Tracking

Event	Purpose
article_view	Article engagement
driver_view	Driver popularity
team_view	Team popularity
article_list	Category / tag tracking


⸻

3.3 Custom Parameters

Configured in GA:
	•	article_id (slug)
	•	article_title
	•	content_type
	•	driver_id
	•	team_id
	•	page_type
	•	tag_name

⸻

4. Consent Mode Implementation (CRITICAL)

4.1 Default State

All users start as:

analytics_storage: 'denied'

👉 No cookies written
👉 No tracking until consent

⸻

4.2 Consent Banner (Frontend)

You MUST implement:

UI:
	•	Accept All
	•	Reject All
	•	Analytics Only

⸻

4.3 On User Action

Accept Analytics:

gtag('consent', 'update', {
  analytics_storage: 'granted'
});


⸻

Accept All:

gtag('consent', 'update', {
  analytics_storage: 'granted',
  ad_storage: 'granted',
  ad_user_data: 'granted',
  ad_personalization: 'granted'
});


⸻

Reject:

gtag('consent', 'update', {
  analytics_storage: 'denied'
});


⸻

4.4 Persist Consent

Store in browser:

localStorage.setItem('cookie_consent', 'accepted');

On load:

const consent = localStorage.getItem('cookie_consent');

if (consent === 'accepted') {
  gtag('consent', 'update', {
    analytics_storage: 'granted'
  });
}


⸻

5. Cookies Used

5.1 Google Analytics Cookies

Cookie	Purpose
_ga	User identification
ga*	Session tracking


⸻

5.2 Your Site Cookies (MVP)

👉 NONE currently

⸻

6. Privacy Policy Requirements

You MUST include:
	•	Analytics usage
	•	Cookie explanation
	•	Consent mechanism
	•	Data retention (GA default)
	•	Third-party services (Google)

⸻

7. Future Architecture (POST-MVP)

⸻

7.1 User Accounts (Planned)

Recommended:

👉 Use Amazon Cognito

⸻

7.2 Data You Will Collect

Data	Purpose
Email	login
Username	identity
Password	auth (hashed)
Comments	user content
Likes	engagement


⸻

7.3 New Cookies (Future)

Cookie	Type
session_token	required
auth_token	required

👉 These are STRICTLY NECESSARY cookies

⸻

7.4 Consent Impact

Cookie Type	Consent Needed
Analytics	YES
Ads	YES
Auth/session	NO (necessary)


⸻

7.5 GA Changes After Login System

Add:

gtag('event', 'login', {
  method: 'email'
});

gtag('event', 'like_article', {
  article_id: id
});

gtag('event', 'comment_posted', {
  article_id: id
});


⸻

8. Security Considerations
	•	Do NOT send PII to GA
	•	Do NOT send emails/usernames
	•	Use anonymized IDs only

⸻

9. Compliance Strategy

MVP:

✔ Basic consent banner
✔ GA consent mode
✔ Privacy policy

Post-MVP:

✔ Full cookie management
✔ Terms & Conditions update
✔ User data policy

⸻

10. Implementation Checklist

MVP (NOW)
	•	GA script added
	•	trackPage implemented
	•	custom events implemented
	•	consent default = denied
	•	consent banner added
	•	localStorage persistence
	•	privacy policy page

⸻

POST-MVP
	•	user authentication (Cognito)
	•	session cookies
	•	comment system
	•	like system
	•	updated consent categories
	•	updated privacy policy

⸻

11. Final Architecture Summary

Flow:

User visits site
→ Consent = denied
→ Banner shown
→ User accepts
→ GA enabled
→ Events tracked

⸻

✅ Final Notes
	•	Current setup = compliant for MVP
	•	Future system = scalable
	•	No rework needed later

⸻

🚀 Next Step

👉 Implement cookie banner UI (ask me, I’ll give exact code)