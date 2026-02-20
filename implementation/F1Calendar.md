Perfect. We go step-by-step, and right now I will only give you:
	•	✅ Final table format (storage design)
	•	✅ The exact 3 APIs we will use
	•	❌ No data yet
	•	❌ No Lambda yet
	•	❌ No API Gateway config yet

Just structure confirmation.

⸻

✅ ARCH-014 — F1 Calendar Architecture (Locked Design)

🎯 Your Functional Requirements (Confirmed)

You need:
	1.	Top panel widget
	•	Shows Next Session only
	•	Countdown timer
	•	Grand Prix name + flag
	•	Session type (FP1 / Sprint / Race etc.)
	2.	Calendar grid page
	•	Shows all 24 races (cards)
	•	Minimal data (round, GP name, country, date)
	3.	Race weekend detail page
	•	Shows all sessions for selected round
	•	Correct order depending on sprint weekend or normal weekend

So yes — we need exactly 3 APIs.

⸻

✅ FINAL TABLE DESIGN (LOCKED)

Table Name

F1Calendar


⸻

🔑 Primary Key Structure

PK  = season          → "2026"
SK  = startTimeUtc    → ISO timestamp string

Example:

PK = "2026"
SK = "2026-03-06T12:30:00Z"

Why this structure?
	•	Naturally sortable
	•	Perfect for “next session” query
	•	No need for composite sessionKey
	•	Cleaner than previous version

⸻

📦 Attributes per Item

Each session is one item.

{
  "season": "2026",
  "round": 1,
  "grandPrix": "Australian Grand Prix",
  "officialName": "FORMULA 1 QATAR AIRWAYS AUSTRALIAN GRAND PRIX 2026",
  "country": "Australia",
  "sessionType": "PRACTICE_1",
  "startTimeUtc": "2026-03-06T12:30:00Z",
  "endTimeUtc": "2026-03-06T13:30:00Z",
  "isSprintWeekend": false
}


⸻

🔍 Why One Item Per Session?

Because it allows:

Next Session Query

PK = "2026"
SK > currentTimeUtc
Limit 1

→ Done.

Race Weekend Query

Filter by:

PK = "2026"
round = X

Full Calendar

Query:

PK = "2026"

Group by round in Lambda.

⸻

✅ THE 3 APIs (FINAL NAMES)

All under /api

⸻

1️⃣ Next Session

GET /api/calendar/next-session

Returns:
	•	round
	•	grandPrix
	•	country
	•	sessionType
	•	startTimeUtc
	•	endTimeUtc
	•	isSprintWeekend

Used only for:
	•	Top widget
	•	Countdown timer

⸻

2️⃣ Full Calendar (Grid View)

GET /api/calendar

Returns:
	•	season
	•	list of 24 rounds
	•	each round:
	•	round number
	•	grandPrix
	•	country
	•	raceDate (RACE session start time)
	•	isSprintWeekend

This API does NOT return all sessions.
Only race-level summary.

⸻

3️⃣ Race Weekend Detail

GET /api/calendar/{round}

Example:

GET /api/calendar/1
GET /api/calendar/12

Returns:
	•	round
	•	grandPrix
	•	country
	•	sessions (ordered array)
	•	PRACTICE_1
	•	PRACTICE_2
	•	PRACTICE_3
	•	SPRINT_QUALIFYING
	•	SPRINT
	•	QUALIFYING
	•	RACE

Frontend renders depending on sessionType.

⸻

✅ Architecture Summary (Calendar Scope Only)

Component	Purpose
F1Calendar table	Stores session-level records
1 Lambda	Handles all 3 APIs
API Gateway	Routes /api/calendar/*
CloudFront	Edge protection
Lambda Authorizer	Token validation


⸻

🔒 ARCHITECTURE STATUS

Calendar system design: LOCKED
	•	One table
	•	One Lambda
	•	Three APIs
	•	Session-level storage
	•	Sorted by time
	•	Sprint flag included

⸻