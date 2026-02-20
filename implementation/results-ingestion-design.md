Perfect.
We will now produce a clean architectural design draft for:

🔹 Manual Result Ingestion System (Qualifying + Race)
🔹 Stored inside F1Races.results
🔹 No extra tables
🔹 No over-normalization
🔹 Admin-only internal utility page

You can save this as results-ingestion-design.md.

⸻

📘 F1 Results Ingestion & Storage Design (MVP)

⸻

1️⃣ Objective

Provide a manual ingestion workflow to:
	•	Paste raw qualifying classification
	•	Paste raw race classification
	•	Convert to structured JSON
	•	Store inside F1Races.results

System must:
	•	Support all 22 drivers
	•	Support DNF / DSQ
	•	Support fastest lap
	•	Support grid position
	•	Allow automatic championship calculation later

⸻

2️⃣ Storage Model (Locked)

Table: F1Races

Each item already contains:

raceId
round
raceDateUtc
...

We add:

results: {
  qualifying: [],
  race: []
}


⸻

3️⃣ Final Results JSON Schema

3.1 Qualifying Structure

{
  "position": 1,
  "driverId": "max_verstappen",
  "teamId": "red_bull",
  "gridPosition": 1,
  "q1": "1:18.234",
  "q2": "1:17.890",
  "q3": "1:17.456"
}

Required Fields
	•	position (1–22)
	•	driverId
	•	teamId
	•	gridPosition

Optional
	•	q1
	•	q2
	•	q3

⸻

3.2 Race Structure

{
  "position": 1,
  "driverId": "max_verstappen",
  "teamId": "red_bull",
  "points": 25,
  "status": "Finished",
  "fastestLap": false,
  "gridPosition": 1
}

Required
	•	position
	•	driverId
	•	teamId
	•	status
	•	points

Optional
	•	fastestLap (boolean)
	•	gridPosition (copied from qualifying if exists)

⸻

4️⃣ Points Policy (Locked)

Points will be auto-calculated inside parser.

Standard FIA scoring:

[25, 18, 15, 12, 10, 8, 6, 4, 2, 1]

	•	Only top 10 receive points
	•	Fastest lap +1 (if top 10 and flag true)
	•	Others = 0

We store points explicitly in JSON.

We DO NOT calculate standings dynamically per request.

⸻

5️⃣ Admin Workflow

Step 1 – Select Race

Dropdown:
	•	Season (default 2026)
	•	Race (by round or raceId)

⸻

Step 2 – Select Result Type

Dropdown:
	•	Qualifying
	•	Race

⸻

Step 3 – Paste Raw Text

Textarea:
	•	Raw copied classification
	•	From image OCR or manual transcription

⸻

Step 4 – Parse (Preview Only)

Frontend calls:

POST /admin/parse-results

Input:

{
  "raceId": "australia-2026",
  "type": "qualifying",
  "rawText": "P1 Max Verstappen ..."
}

Lambda does:
	•	Split lines
	•	Extract position
	•	Extract full driver name
	•	Map full name → driverId (from F1Drivers table)
	•	Map driverId → teamId
	•	Extract times/status
	•	Calculate points (if race)
	•	Return structured JSON

Response:

{
  "parsedResults": [...]
}


⸻

Step 5 – Confirm & Save

Frontend shows preview table.

On confirm:

POST /admin/save-results

Input:

{
  "raceId": "australia-2026",
  "type": "race",
  "results": [...]
}

Lambda updates:

F1Races.results.race = [...]

No overwrite of qualifying unless selected.

⸻

6️⃣ DynamoDB Update Strategy

Use:

UpdateExpression:
SET results.#type = :data

ExpressionAttributeNames:

"#type": "race"

or

"#type": "qualifying"

This ensures:
	•	Only selected subfield replaced
	•	No other race metadata touched

⸻

7️⃣ Standings Calculation Strategy (Future)

Later we create:

GET /api/standings/drivers
GET /api/standings/constructors

Logic:

for each race in season:
  for each race.results.race:
     driverPoints[driverId] += result.points
     teamPoints[teamId] += result.points

Sort descending.

No extra table required for MVP.

Optional later: snapshot standings table for performance.

⸻

8️⃣ Validation Rules

Parser must:
	•	Ensure exactly 22 drivers returned
	•	Ensure no duplicate driverId
	•	Ensure position unique
	•	Ensure points valid
	•	Reject if unknown driver name

This prevents corrupted standings.

⸻

9️⃣ Security

Admin endpoints:
	•	Not exposed on public CF behavior
	•	Or protected by secret header:

X-Admin-Key



No authentication system required for MVP.

⸻

🔟 What We Are NOT Doing (MVP)
	•	No sprint results storage
	•	No lap charts
	•	No pit stop analytics
	•	No telemetry
	•	No separate results table

All inside F1Races.

⸻

1️⃣1️⃣ Architectural Advantages
	•	Simple
	•	Deterministic
	•	Minimal tables
	•	No over-engineering
	•	Easy to patch
	•	Easy to extend later

⸻

🔒 Architecture Lock Summary

We now have:
	•	F1Calendar → session-level timing
	•	F1Races → metadata + results
	•	F1Drivers → driver registry
	•	F1Teams → team registry

And a clean ingestion flow.