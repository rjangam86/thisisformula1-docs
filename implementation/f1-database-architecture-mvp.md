Good.
We’re now freezing the database layer definition for MVP.

Below is the authoritative Database Architecture Document.
Save this as:

f1-database-architecture-mvp.md

This reflects your current implementation state + required GSIs.

⸻

📘 F1 Editorial Website

Database Architecture (MVP – Read Only Public)

⸻

1️⃣ Overview

This database design supports:
	•	Calendar (session-level)
	•	Race metadata
	•	Qualifying & race results
	•	Drivers
	•	Teams
	•	Standings (calculated dynamically)
	•	Future multi-season support

MVP is read-only for public.
Admin ingestion updates F1Races.results.

⸻

2️⃣ Table: F1Calendar

Purpose

Stores session-level timing data (FP1, FP2, Quali, Race, Sprint).

This is the authoritative time source.

⸻

Primary Key

Attribute	Type	Role
season	String	Partition Key
startTimeUtc	String (ISO)	Sort Key

Example:

season = "2026"
startTimeUtc = "2026-03-08T04:00:00Z"


⸻

Attributes

Field	Type	Notes
season	String	e.g. “2026”
round	Number	1–24
sessionType	String	“Practice_1”, “Qualifying”, “Race”
startTimeUtc	String	ISO 8601 UTC
circuitName	String	e.g. “Albert Park”
country	String	e.g. “Australia”
timezone	String	e.g. “Australia/Melbourne”
grandPrixName	String	Short GP name


⸻

Indexes

None required.

Query patterns:
	•	All sessions for season
	•	Next session (startTimeUtc > now)
	•	Race sessions filtered by sessionType=“Race”

⸻

3️⃣ Table: F1Races

Purpose

Stores race-level metadata + qualifying & race results.

Exactly 1 row per Grand Prix.

⸻

Primary Key

Attribute	Type	Role
raceId	String	Partition Key

Example:

australia-2026


⸻

Attributes

Field	Type	Notes
raceId	String	Unique
season	String	“2026”
round	Number	1–24
raceDateUtc	String	Must match F1Calendar Race session
grandPrixName	String	Short name
officialName	String	Full sponsored name
country	String	
circuitName	String	
timezone	String	
flagEmoji	String	🇦🇺
circuitImageUrl	String	
results	Map	See below


⸻

Results Structure

results: {
  qualifying: [
    {
      "position": 1,
      "driverId": "max_verstappen",
      "teamId": "red_bull",
      "gridPosition": 1,
      "q1": "1:18.234",
      "q2": "1:17.890",
      "q3": "1:17.456"
    }
  ],
  race: [
    {
      "position": 1,
      "driverId": "max_verstappen",
      "teamId": "red_bull",
      "points": 25,
      "status": "Finished",
      "fastestLap": false,
      "gridPosition": 1
    }
  ]
}


⸻

Required GSI

GSI1 – SeasonRoundIndex

Attribute	Role
season	Partition Key
round	Sort Key

Purpose
	•	Fetch all races by season
	•	Sort by round
	•	Retrieve next race
	•	Multi-season support later

⸻

4️⃣ Table: F1Drivers

Purpose

Driver registry + static career data.

⸻

Primary Key

Attribute	Role
driverId	Partition Key

Example:

max_verstappen


⸻

Attributes

Field	Type
driverId	String
fullName	String
number	Number
nationality	String
dateOfBirth	String
firstGrandPrix	String
lastGrandPrix	String
podiums	Number
wins	Number
worldChampionships	Number
teamId	String
photoUrl	String


⸻

Required GSI

GSI1 – TeamDriversIndex

Attribute	Role
teamId	Partition Key

Purpose

Fetch all drivers for a team page.

⸻

5️⃣ Table: F1Teams

Purpose

Team registry + metadata.

⸻

Primary Key

Attribute	Role
teamId	Partition Key

Example:

red_bull


⸻

Attributes

Field	Type
teamId	String
shortName	String
fullName	String
country	String
teamPrincipal	String
technicalDirector	String
chassis	String
powerUnit	String
logoUrl	String
carImageUrl	String


⸻

Indexes

None required.

⸻

6️⃣ Optional Table: F1Seasons (Recommended)

Purpose

Removes hardcoding of season.

⸻

Primary Key

Attribute	Role
season	Partition Key


⸻

Attributes

Field	Type
season	String
totalRounds	Number
status	String (“active” / “completed”)
currentRound	Number
lastUpdatedUtc	String


⸻

7️⃣ Standings Strategy (MVP)

No separate table required.

Standings calculated dynamically:

for race in F1Races where season=2026:
  for result in race.results.race:
    driverPoints[driverId] += points
    teamPoints[teamId] += points

Sorted descending.

Snapshot table may be added later.

⸻

8️⃣ Security Architecture (Summary)

Public:
	•	Read-only APIs

Admin:
	•	Protected endpoints
	•	Update only F1Races.results
	•	No public write access

⸻

9️⃣ Final GSI List To Add

You need to add:

⸻

1️⃣ F1Races → SeasonRoundIndex

Partition Key:

season

Sort Key:

round


⸻

2️⃣ F1Drivers → TeamDriversIndex

Partition Key:

teamId


⸻

No other GSIs required for MVP.

⸻

🔒 MVP Database Status

After adding:
	•	GSI on F1Races
	•	GSI on F1Drivers

Your database layer is:
	•	Clean
	•	Scalable
	•	Multi-season ready
	•	Not over-normalized
	•	Ready for standings

    