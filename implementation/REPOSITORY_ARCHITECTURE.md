📄 1️⃣ REPOSITORY_ARCHITECTURE.md (Updated & Final)

# ThisIsFormula1 – Repository Architecture (Final MVP Structure)

Status: Operational Architecture Locked
Owner: Rahul

---

# High-Level System Separation

We maintain THREE independent systems:

1. Public Website (Frontend Only)
2. Public Website API (Read-Only Backend)
3. Media Automation Pipeline (Internal System)

Each has its own repository.
Each deploys independently.
Each has its own lifecycle.

---

# REPO 1 — thisisformula1-web

Purpose:
Public website frontend only.

Contains:
- Static HTML
- CSS
- JS
- Public UI pages
- API integration code
- Deployment scripts

Structure:

/web
  index.html
  /pages
  /components
  /assets
  /styles
  /scripts

/docs
  API_CONTRACT.md
  UI_SPEC.md
  SEO_PLAN.md

/scripts
  deploy-dev.sh
  deploy-prod.sh

README.md

Rules:
- No Lambda code here
- No database schema here
- No pipeline logic here


⸻

REPO 2 — thisisformula1-api

Purpose:
Public read-only backend.

Contains:
	•	Lambda functions
	•	Lambda Authorizer
	•	Standings engine
	•	DB patch scripts
	•	Security enforcement logic

Structure:

/lambdas
/calendar
/races
/teams
/drivers
/standings
/health

/authorizer
index.js

/utils
hmac.js
validation.js

/db
/schemas
/patches
DATABASE_SCHEMA.md

/docs
API_CONTRACT.md
SECURITY_IMPLEMENTATION.md

/scripts
deploy-dev.sh
deploy-prod.sh

README.md

Rules:
	•	No frontend code
	•	No media pipeline logic
	•	Only public API functionality

---

# REPO 3 — thisisformula1-media-pipeline

Purpose:
Internal content automation + admin system.

Contains:
- Article generator Lambdas
- SQS handlers
- Social posting logic
- Telegram integration
- Image processing
- Internal GUIs

Structure:

/lambdas
  /article-generation
  /scheduler
  /sqs-consumers
  /social-posting
  /telegram
  /image-processing

/web
  /dashboard
  /jobs
  /library
  /stories
  /scheduler
  index.html

/db
  DATABASE_SCHEMA.md
  SQS_FLOW.md

/docs
  PIPELINE_ARCHITECTURE.md

/scripts
  deploy-dev.sh
  deploy-prod.sh

README.md

Rules:
- Never mix website API logic here
- Internal only
- Not publicly accessible


⸻

Documentation Rule (Very Important)

Each repository must contain its own:
	•	README.md
	•	/docs folder
	•	DATABASE_SCHEMA.md (if DB used)
	•	SECURITY.md (if applicable)

Do NOT store all documentation in one mega repository.

Keep documentation close to the code it governs.

⸻

Backup Strategy

GitHub = Source of Truth.

S3 is not source of truth.
AWS Console is not source of truth.

⸻

Versioning Rule

Each repository tags releases independently:

release-YYYY-MM-DD

---

# 📄 2️⃣ DEVELOPMENT_WORKFLOW.md (Updated)

```md
# Development Workflow – ThisIsFormula1

Status: MVP Manual Deployment

---

# Local Requirements

Install:

- Node.js >=18
- AWS CLI v2
- Git
- GitHub CLI (optional)

Configure AWS profiles for:
- dev
- prod

---

# Repo-Based Development

You NEVER edit code directly in S3.

Always:

1. Pull latest from Git
2. Develop locally
3. Commit
4. Push
5. Deploy from local

---

# Web Repo Workflow

git checkout develop
make changes
git commit -m "UI: teams grid"
git push origin develop

Deploy to DEV:

aws s3 sync ./web s3://dev-bucket --delete
aws cloudfront create-invalidation --distribution-id DEV_ID --paths "/*"

Test.

Promote:

git checkout main
git merge develop
git tag release-YYYY-MM-DD
git push --tags

Deploy to PROD:

aws s3 sync ./web s3://prod-bucket --delete
aws cloudfront create-invalidation --distribution-id PROD_ID --paths "/*"


⸻

API Repo Workflow

git checkout develop
edit lambda
zip function.zip .
aws lambda update-function-code \
  --function-name dev-function \
  --zip-file fileb://function.zip

Test.

Promote via main branch + tag.


⸻

Media Pipeline Workflow

Same structure as API.

Never deploy directly from console.
Always deploy from repo.

---

# 📄 3️⃣ DEPLOYMENT_RUNBOOK.md (Updated)

```md
# Production Deployment Runbook

Status: MVP Manual

---

# Web Production

Pre-check:
- Security implemented?
- WAF enabled?
- Release tag created?

Deploy:

aws s3 sync ./web s3://prod-bucket --delete
aws cloudfront create-invalidation --distribution-id PROD_ID --paths "/*"

Verify:
- Homepage
- Teams
- Drivers
- Calendar
- Standings

---

# API Production

Deploy each lambda:

aws lambda update-function-code \
  --function-name prod-function \
  --zip-file fileb://dist.zip

Verify:
- /health
- /teams
- /drivers
- /calendar
- /standings

---

# Rollback

git checkout previous-release-tag

Redeploy S3 + Lambda.

Invalidate CloudFront.


⸻