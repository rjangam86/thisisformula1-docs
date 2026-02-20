# Development Workflow – ThisIsFormula1

Status: MVP Workflow (Manual Deployment)

---

# 1. Local Setup Requirements

Install:

- Node.js (>=18)
- AWS CLI v2
- Git
- GitHub CLI (optional)
- VS Code

Configure AWS:

aws configure

Set default profile for dev region.

---

# 2. Web Development Flow

Step 1:
Work locally in thisisformula1-web repo.

Step 2:
Test locally (optional):

python -m http.server 8080

Step 3:
Commit changes:

git add .
git commit -m "UI: teams grid update"

Step 4:
Push to develop:

git push origin develop

Step 5:
Deploy to dev bucket:

aws s3 sync ./web s3://<dev-bucket> --delete

Step 6:
Invalidate dev CloudFront:

aws cloudfront create-invalidation \
--distribution-id <DEV_ID> \
--paths "/*"

Test on dev domain.

---

# 3. API Development Flow

Step 1:
Work in thisisformula1-api repo.

Step 2:
Modify Lambda code.

Step 3:
Build zip:

zip -r function.zip .

Step 4:
Deploy to dev:

aws lambda update-function-code \
--function-name <dev-function-name> \
--zip-file fileb://function.zip

Step 5:
Test via dev API endpoint.

---

# 4. Promotion to Production

Never edit prod manually.

When ready:

git checkout main
git merge develop
git tag release-YYYY-MM-DD
git push --tags

Deploy same commit to prod bucket and prod Lambda.