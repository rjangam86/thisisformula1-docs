Perfect 👍
Below is your full implementation documentation file.
You can paste this into:

docs/IMPLEMENTATION_MIGRATION_PHASE1.md

or similar.

⸻

ThisIsFormula1 — Phase 1 Infrastructure Restructuring Implementation

Date: 2026
Owner: Rahul Jangam
Scope: Repository split, deployment standardization, local dev stabilization, and operational structure setup.

⸻

1️⃣ Objective of This Phase

This restructuring phase was executed to:
	•	Split monolithic structure into clean repositories
	•	Standardize Lambda deployment
	•	Standardize S3 + CloudFront deployment
	•	Fix local development without hacks (no symlinks)
	•	Introduce branch strategy
	•	Introduce deployment scripts
	•	Prepare production alignment
	•	Prepare for CI/CD integration
	•	Enable clean developer onboarding

This phase does NOT include final production hardening or API Gateway alignment.

⸻

2️⃣ Repository Restructuring

Repositories Created
	1.	thisisformula1-web
	2.	thisisformula1-api
	3.	thisisformula1-media-pipeline
	4.	thisisformula1-docs

⸻

Folder Structure (Web Repo)

docs/
scripts/
web/
LICENSE
.gitignore


⸻

Folder Structure (API / Media Pipeline)

Each Lambda has its own folder:

LambdaReadArticles/
LambdaGeneratePresignedURLs/
LambdaJobDistributionHandler/
...
scripts/
.gitignore


⸻

3️⃣ Branch Strategy

For each repository:
	•	main → stable reference
	•	develop → active development branch

Branch creation process:

git checkout -b develop
git push -u origin develop

All active work now continues on develop.

⸻

4️⃣ Local Development Fix (CRITICAL)

Problem Encountered
	•	Website originally had index.html inside /html
	•	Relative path conflicts:
	•	js/...
	•	../js/...
	•	Attempted fix using symlinks (ln -s)
	•	This caused recursive folder duplication:
	•	/html/html/html/...
	•	/upload/upload/upload/...
	•	S3 bucket became polluted (~400MB recursive uploads)

⸻

Root Cause

Symlinked directories were recursively synced via:

aws s3 sync ./web s3://bucket --delete

aws s3 sync followed symlinks → infinite recursion.

⸻

Final Fix (Correct Architecture)

Reverted to original template structure:

index.html (ROOT)
css/
js/
images/
upload/
html/partials/

All references updated to absolute root paths:

Before:

../js/script.js

After:

/js/script.js


⸻

Result
	•	Python local server works
	•	No symlinks required
	•	No recursive folders
	•	Clean S3 sync
	•	Clean CloudFront deploy

⸻

5️⃣ Local Dev Server

Used:

python3 -m http.server 8001

Run from inside:

thisisformula1-web/web

All pages tested:
	•	index
	•	category
	•	tag
	•	single-post
	•	partial header/footer

All returned HTTP 200.

⸻

6️⃣ S3 Deployment Script

File:

scripts/deploy-dev.sh

#!/bin/bash
set -euo pipefail

BUCKET="dev.thisisformulaone.net"
PROFILE="dev"
DISTRIBUTION_ID="E13MKW62YZKMM9"

echo "Syncing to S3..."
aws s3 sync ./web "s3://$BUCKET" --delete --profile "$PROFILE"

echo "Invalidating CloudFront..."
aws cloudfront create-invalidation \
  --distribution-id "$DISTRIBUTION_ID" \
  --paths "/*" \
  --profile "$PROFILE"

echo "Deploy complete."


⸻

CloudFront Invalidation Purpose

CloudFront caches content globally.

Invalidation command:

aws cloudfront create-invalidation \
  --distribution-id <ID> \
  --paths "/*"

This forces CloudFront to refresh cached files after deployment.

⸻

7️⃣ Lambda Deployment Standardization

Script:

scripts/deploy-lambda.sh

#!/bin/bash
set -euo pipefail

if [ $# -lt 3 ]; then
  echo "Usage: ./deploy-lambda.sh <lambda-folder> <lambda-name-aws> <profile>"
  exit 1
fi

FOLDER=$1
LAMBDA_NAME=$2
PROFILE=$3

cd "$FOLDER"

npm install --omit=dev

rm -f lambda.zip
zip -r lambda.zip . -x "*.git*" -x "node_modules/.cache/*"

aws lambda update-function-code \
  --function-name "$LAMBDA_NAME" \
  --zip-file fileb://lambda.zip \
  --profile "$PROFILE"

rm lambda.zip

echo "Deployment complete."


⸻

Deployment Usage

From repository root:

./scripts/deploy-lambda.sh LambdaReadArticles TIF1-ReadArticles dev


⸻

Region Resolution

Region is automatically determined by:

~/.aws/config profile

Example:

[profile dev]
region = ap-southeast-2

[profile prod]
region = us-east-1


⸻

8️⃣ Git Hygiene

.gitignore includes:

node_modules/
.DS_Store
lambda.zip

Zip files are never committed.

Node modules are not committed.

⸻

9️⃣ Production Preparation Status

Production Lambda created in:
	•	Region: us-east-1
	•	Using global DynamoDB table
	•	IAM role reused (temporary)
	•	Code deployment verified

API Gateway + CloudFront alignment pending.

⸻

🔟 Remaining Work (Tracked via Tickets)
	•	TKT-020 — Production API Gateway Alignment
	•	TKT-021 — Lambda Production Config Audit
	•	TKT-023 — Security Hardening
	•	TKT-024 — CI/CD Automation

⸻

11️⃣ Lessons Learned
	1.	Never use symlinks in web folders before S3 sync.
	2.	Always use root-based paths in static sites.
	3.	Never sync before dry run:

aws s3 sync ./web s3://bucket --dryrun


	4.	Always test locally before deploy.
	5.	Always keep clean repository as source of truth.

⸻

12️⃣ Current System State

✅ Repository structure clean
✅ Web deploy standardized
✅ Lambda deploy standardized
✅ Dev environment stable
✅ Local development stable
⏳ Production alignment pending
⏳ CI/CD pending

⸻

Phase 1 Status: COMPLETE

This phase successfully transitioned the project from experimental infrastructure to structured, deployable, and maintainable architecture.