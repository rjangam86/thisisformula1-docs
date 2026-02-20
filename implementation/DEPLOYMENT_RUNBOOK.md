# Production Deployment Runbook

Status: MVP Manual Deployment

---

# 1. Web Production Deployment

Pre-check:
- All changes merged into main
- Release tag created

Deploy:

aws s3 sync ./web s3://<prod-bucket> --delete

Invalidate CloudFront:

aws cloudfront create-invalidation \
--distribution-id <PROD_ID> \
--paths "/*"

Verify:
- Homepage loads
- API calls work
- No console errors

---

# 2. API Production Deployment

Build zip.

Deploy:

aws lambda update-function-code \
--function-name <prod-function-name> \
--zip-file fileb://function.zip

Test:
- Health endpoint
- Teams endpoint
- Drivers endpoint

---

# 3. Rollback Procedure

If failure:

git checkout <previous-release-tag>

Redeploy to S3 and Lambda.

Invalidate CloudFront again.

---

# 4. Security Activation Checklist

Before launch:

- Enable WAF on prod
- Enable rate limiting
- Enable Lambda Authorizer
- Enable CloudFront cookie injection
- Confirm Secrets Manager configured