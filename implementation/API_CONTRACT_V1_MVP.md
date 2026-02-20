Perfect.
We are freezing the Public API Contract v1 (MVP) including:
	•	Existing article APIs (already implemented ✅)
	•	New calendar / race / team / driver / standings APIs
	•	Health check endpoint

Below is your ready-to-save Markdown file.

⸻

📄 API_CONTRACT_V1_MVP.md

⸻

ThisIsFormula1 – Public API Contract

Version: v1
Status: Design Frozen – Ready for Implementation

Base Path:

/api/v1

All endpoints:
	•	GET only
	•	JSON response
	•	Protected by HMAC + CloudFront (production)
	•	No wildcard CORS
	•	Rate limited

⸻

0️⃣ Health Check

GET /api/v1/health

Purpose:
Infrastructure health monitoring (CloudFront, ALB, uptime monitor)

Response:

{
  "status": "ok",
  "service": "thisisformula1-api",
  "version": "v1",
  "timestamp": "2026-03-01T10:00:00Z"
}

Must not query DynamoDB.

⸻

1️⃣ Articles (Already Implemented)

1.1 GET /api/v1/articles

Query:
	•	limit (max 50)
	•	cursor
	•	type
	•	tag

⸻

1.2 GET /api/v1/articles/{articleId}

⸻

1.3 GET /api/v1/tags

⸻

1.4 GET /api/v1/articles/type/{type}

⸻

2️⃣ Calendar & Race APIs

⸻

2.1 Next Session

GET /api/v1/calendar/next

Logic:
	•	Query F1 calendar
	•	Filter season = current season
	•	startTimeUtc > now()
	•	Return earliest session

Response:

{
  "season": "2026",
  "round": 5,
  "raceId": "saudi-2026",
  "grandPrixName": "Saudi Arabian Grand Prix",
  "country": "Saudi Arabia",
  "countryCode": "SA",
  "sessionType": "PRACTICE_1",
  "startTimeUtc": "2026-04-17T20:30:00Z",
  "timezone": "Asia/Riyadh"
}


⸻

2.2 Full Season Calendar (Grid)

GET /api/v1/calendar?season=2026

Response:

{
  "season": "2026",
  "races": [
    {
      "round": 1,
      "raceId": "australia-2026",
      "grandPrixName": "Australian Grand Prix",
      "country": "Australia",
      "raceDateUtc": "2026-03-08T04:00:00Z",
      "circuitImageUrl": "..."
    }
  ]
}


⸻

2.3 Race Weekend Detail

GET /api/v1/races/{raceId}

Response:

{
  "raceId": "saudi-2026",
  "season": "2026",
  "round": 5,
  "grandPrixName": "Saudi Arabian Grand Prix",
  "timezone": "Asia/Riyadh",
  "sessions": [
    {
      "sessionType": "PRACTICE_1",
      "startTimeUtc": "..."
    }
  ],
  "results": {
    "qualifying": [],
    "race": []
  }
}


⸻

3️⃣ Teams

⸻

3.1 GET /api/v1/teams

Returns basic list:

{
  "items": [
    {
      "teamId": "redbull",
      "name": "Red Bull Racing",
      "logoUrl": "...",
      "carImageUrl": "..."
    }
  ]
}


⸻

3.2 GET /api/v1/teams/{teamId}

Returns full detail:

{
  "teamId": "redbull",
  "name": "Red Bull Racing",
  "teamPrincipal": "Laurent Mekies",
  "technicalDirector": "Pierre Waché",
  "chassis": "RB22",
  "drivers": [
    {
      "driverId": "verstappen",
      "name": "Max Verstappen",
      "nationality": "Netherlands",
      "photoUrl": "..."
    }
  ]
}

Driver data joined via:
currentTeamId (GSI index)

⸻

4️⃣ Drivers

⸻

4.1 GET /api/v1/drivers

Optional filter:

?teamId=redbull

Response:

{
  "items": [
    {
      "driverId": "verstappen",
      "name": "Max Verstappen",
      "nationality": "Netherlands",
      "number": 1,
      "currentTeamId": "redbull",
      "photoUrl": "..."
    }
  ]
}


⸻

4.2 GET /api/v1/drivers/{driverId}

Response:

{
  "driverId": "verstappen",
  "name": "Max Verstappen",
  "dateOfBirth": "1997-09-30",
  "nationality": "Netherlands",
  "number": 1,
  "career": {
    "firstGrandPrix": "Australia 2015",
    "wins": 55,
    "podiums": 90,
    "worldChampionships": 3
  },
  "currentTeamId": "redbull",
  "photoUrl": "..."
}


⸻

5️⃣ Standings

⸻

5.1 GET /api/v1/standings/drivers?season=2026

Response:

{
  "season": "2026",
  "updatedAfterRound": 4,
  "items": [
    {
      "position": 1,
      "driverId": "verstappen",
      "points": 88,
      "wins": 2
    }
  ]
}


⸻

5.2 GET /api/v1/standings/constructors?season=2026

Response:

{
  "season": "2026",
  "updatedAfterRound": 4,
  "items": [
    {
      "position": 1,
      "teamId": "redbull",
      "points": 150,
      "wins": 3
    }
  ]
}


⸻

🔒 Security Enforcement

All endpoints must:
	•	Require valid CloudFront session cookie (production)
	•	Validate HMAC header via Lambda Authorizer
	•	Enforce max limit caps
	•	Reject unknown query params
	•	Reject wildcard origin
	•	Apply rate limiting via WAF (production)

⸻

🧱 Total Public Endpoints

Category	Count
Health	1
Articles	4
Calendar	3
Teams	2
Drivers	2
Standings	2

Total: 14 endpoints

⸻

🚦 Status

API Contract: Frozen
Ready for Implementation Phase

⸻