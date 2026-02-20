```md
# 11 — Error Handling & Logging  
_ThisIsFormulaOne Platform — Core Reliability Specification_

This document defines the unified error-handling strategy and structured logging model for all Lambda functions and API Gateway integrations.

---

# 1. Goals

- Ensure backend operations fail **predictably**
- Provide **clear, structured logs** for fast debugging
- Maintain **uniform JSON error envelopes**
- Prevent unintentional data leaks (no stack traces)
- Enable CloudWatch monitoring and alerting consistency

---

# 2. Standard Error Response Envelope

All API errors MUST return this format:

```json
{
  "error": {
    "type": "BadRequestError",
    "message": "Missing required parameter: page",
    "status_code": 400
  }
}
```

**Field Definitions**
- `type`: Category of error  
- `message`: Human-readable explanation  
- `status_code`: HTTP status (400, 404, 500, etc.)

---

# 3. Error Categories

## 3.1 Client Errors (4xx)
- `ValidationError`
- `MissingParameterError`
- `NotFoundError`
- `QueryParameterError`

## 3.2 Server Errors (5xx)
- `DynamoDBQueryError`
- `SerializationError`
- `InternalServerError`
- `TimeoutError`

---

# 4. Error Handling Rules

1. Validate all incoming query & path parameters.  
2. Wrap all DynamoDB errors using safe server-side messages.  
3. Never send stack traces to the user.  
4. Apply timeouts to all DB calls (3 seconds via `AbortController`).  
5. Retry once on DynamoDB throttling.  
6. All unknown failures → `InternalServerError`.

---

# 5. Logging Requirements

Logs MUST be structured JSON. Example:

```json
{
  "level": "info",
  "event": "ArticlesFetched",
  "params": { "page": 1, "per_page": 6 },
  "result_count": 6,
  "duration_ms": 143
}
```

Allowed levels:
- `info`
- `warn`
- `error`
- `debug` (disabled in production)

---

# 6. Logging Rules

- DO NOT log:
  - article full content  
  - secrets, tokens, env vars  
  - raw internal errors  

- DO log:
  - query parameters  
  - number of results returned  
  - DynamoDB response metadata  
  - execution duration  
  - error categories (not stack traces)

---

# 7. Example Error Logs

## 7.1 Validation Error

```json
{
  "error": {
    "type": "ValidationError",
    "message": "'page' must be >= 1",
    "status_code": 400
  }
}
```

## 7.2 DynamoDB Failure Log

```json
{
  "level": "error",
  "event": "DynamoDBQueryError",
  "details": {
    "error_message": "ProvisionedThroughputExceededException",
    "params": { "page": 1, "per_page": 6 }
  }
}
```

---

# 8. API Gateway Logging Setup

Enable:
- **Access Logs** (JSON structured)
- **Execution Logs** (INFO only)

Recommended access log format:

```
$context.requestId $context.httpMethod $context.resourcePath $context.status $context.integrationStatus
```

---

# 9. CloudWatch Metrics (Critical)

| Metric | Purpose |
|--------|---------|
| **5XXErrors** | Detect server failures |
| **4XXErrors** | Detect invalid user calls |
| **Latency** | Identify slow DB queries or timeouts |
| **Throttles** | DynamoDB capacity issues |
| **IntegrationLatency** | API → Lambda bottlenecks |

---

# 10. Future Improvements (Phase 2)

- OpenSearch integration for full-text log search  
- S3 archival of CloudWatch logs  
- Real-time dashboards (Grafana/CloudWatch Dashboards)  
- Alerts (SNS Slack/Email) for:
  - >5% error rate  
  - DynamoDB throttles  
  - Latency spikes  

---

# 11. Summary

This file defines:
- Standardized JSON error format  
- Structured logging requirements  
- Clear error categories  
- API Gateway log configuration  
- CloudWatch metrics strategy  

This guarantees predictable behaviour and production-grade observability for all API endpoints.
```