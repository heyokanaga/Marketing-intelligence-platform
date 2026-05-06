# API STANDARDS
## Marketing Intelligence Platform - RESTful API Specification

**Last Updated:** 2026-05-06  
**API Version:** 1.0  
**Status:** Standard Operating Procedures

---

## PURPOSE

Define consistent API contracts across the platform to ensure:
- Predictable client integration
- Clear error handling
- Scalable async operations
- Future-proof versioning

---

## BASE URL

```
Production: https://api.markerintel.com/api/v1
Staging: https://staging-api.markerintel.com/api/v1
Development: http://localhost:8000/api/v1
```

---

## AUTHENTICATION

### Headers

```
Authorization: Bearer <JWT_TOKEN>
Content-Type: application/json
X-Organization-ID: <org-id>  (optional, inferred from token if omitted)
```

### Token Format

```json
{
  "user_id": "uuid",
  "org_id": "uuid",
  "email": "user@company.com",
  "role": "admin",
  "exp": 1234567890,
  "iat": 1234567800
}
```

---

## ENDPOINT NAMING CONVENTIONS

### Pattern

```
[METHOD] /api/v1/[resource]/[id]/[action]
```

### Examples

```
GET    /api/v1/strategies                    # List all strategies
POST   /api/v1/strategies                    # Create strategy
GET    /api/v1/strategies/:id                # Get specific strategy
PUT    /api/v1/strategies/:id                # Update strategy
DELETE /api/v1/strategies/:id                # Delete strategy

POST   /api/v1/strategies/:id/export         # Export strategy
POST   /api/v1/strategies/:id/approve        # Approve strategy
GET    /api/v1/strategies/:id/versions       # Get version history
```

### Resource Naming
- Use plural nouns: `/strategies`, not `/strategy`
- Use kebab-case for compound words: `/market-analysis`
- Hierarchy: `/organizations/:id/brands/:id/strategies`

---

## REQUEST/RESPONSE ENVELOPE

### Success Response

```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "title": "Q2 2026 Strategy",
    ...
  },
  "meta": {
    "timestamp": "2026-05-06T10:30:00Z",
    "version": "1.0"
  }
}
```

### Paginated Response

```json
{
  "success": true,
  "data": [
    { "id": "uuid1", ... },
    { "id": "uuid2", ... }
  ],
  "pagination": {
    "page": 1,
    "per_page": 20,
    "total": 150,
    "total_pages": 8,
    "has_next": true,
    "has_prev": false
  },
  "meta": {
    "timestamp": "2026-05-06T10:30:00Z"
  }
}
```

### Error Response

```json
{
  "success": false,
  "error": {
    "code": "STRATEGY_NOT_FOUND",
    "message": "Strategy with ID xyz not found",
    "status": 404,
    "details": {
      "field": "strategy_id",
      "value": "xyz"
    }
  },
  "meta": {
    "timestamp": "2026-05-06T10:30:00Z",
    "request_id": "req_12345"
  }
}
```

---

## ERROR STRUCTURE

### Standard Errors

```json
{
  "code": "ERROR_CODE",
  "message": "Human readable message",
  "status": 400,
  "details": {
    "field": "value",
    "reason": "specific reason"
  }
}
```

### Common Error Codes

| Code | Status | Meaning |
|------|--------|---------|
| `INVALID_REQUEST` | 400 | Malformed request |
| `VALIDATION_ERROR` | 422 | Request data invalid |
| `UNAUTHORIZED` | 401 | Missing/invalid auth |
| `FORBIDDEN` | 403 | Insufficient permissions |
| `NOT_FOUND` | 404 | Resource doesn't exist |
| `CONFLICT` | 409 | Resource already exists |
| `RATE_LIMITED` | 429 | Too many requests |
| `INTERNAL_ERROR` | 500 | Server error |
| `SERVICE_UNAVAILABLE` | 503 | External service down |

### Validation Error Example

```json
{
  "code": "VALIDATION_ERROR",
  "message": "Request validation failed",
  "status": 422,
  "details": {
    "errors": [
      {
        "field": "title",
        "message": "Title required",
        "value": ""
      },
      {
        "field": "market_opportunity",
        "message": "Must be between 0 and 100",
        "value": 150
      }
    ]
  }
}
```

---

## PAGINATION

### Query Parameters

```
GET /api/v1/strategies?page=1&per_page=20&sort=-created_at
```

**Parameters:**
- `page` (default: 1) - Page number (1-indexed)
- `per_page` (default: 20, max: 100) - Items per page
- `sort` (default: -created_at) - Sort field (prefix with `-` for descending)

### Cursor-Based Pagination (for large datasets)

```
GET /api/v1/strategies?cursor=next_token&per_page=50

Response:
{
  "data": [...],
  "pagination": {
    "cursor": "eyJpZCI6IjEyMzQ1In0=",
    "has_next": true,
    "next_cursor": "eyJpZCI6IjY3ODkwIn0="
  }
}
```

---

## FILTERING & QUERYING

### Query Syntax

```
GET /api/v1/strategies?status=approved&market_id=saas&confidence_gte=0.8

Standard operators:
- eq (equals): field=value
- neq (not equals): field_neq=value
- gt (greater than): field_gt=value
- gte (greater than or equal): field_gte=value
- lt (less than): field_lt=value
- lte (less than or equal): field_lte=value
- in (one of): field_in=value1,value2
- contains (substring): field_contains=substring
```

### Example Complex Query

```
GET /api/v1/strategies?
  status_in=approved,active&
  confidence_gte=0.8&
  created_at_gte=2026-01-01&
  sort=-market_opportunity&
  page=1&
  per_page=50
```

---

## ASYNC JOB OPERATIONS

### Submitting a Long-Running Task

```
POST /api/v1/strategies/async/generate

Request:
{
  "market_id": "enterprise-saas",
  "depth": "deep"
}

Response (202 Accepted):
{
  "success": true,
  "data": {
    "job_id": "job_123456",
    "status": "queued",
    "estimated_wait_seconds": 30
  },
  "meta": {
    "polling_url": "/api/v1/jobs/job_123456"
  }
}
```

### Polling for Job Status

```
GET /api/v1/jobs/job_123456

Response (while processing):
{
  "success": true,
  "data": {
    "job_id": "job_123456",
    "status": "processing",
    "progress": 65,
    "estimated_completion_seconds": 15
  }
}

Response (complete):
{
  "success": true,
  "data": {
    "job_id": "job_123456",
    "status": "complete",
    "result": {
      "strategy_id": "strat_789",
      "title": "Q2 Enterprise Strategy"
    }
  }
}
```

### Webhook for Completion (Future)

```
POST https://customer.com/webhooks/job-complete

{
  "event": "job.complete",
  "job_id": "job_123456",
  "result": { ... },
  "timestamp": "2026-05-06T10:45:00Z",
  "signature": "hmac_sha256_signature"
}
```

---

## RATE LIMITING

### Headers

```
Request:
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1620000000

Response (429 Too Many Requests):
{
  "success": false,
  "error": {
    "code": "RATE_LIMITED",
    "message": "API rate limit exceeded",
    "status": 429,
    "details": {
      "limit": 1000,
      "reset_at": "2026-05-06T11:00:00Z",
      "retry_after_seconds": 30
    }
  }
}
```

### Limits per Organization

| Tier | Requests/min | Requests/day |
|------|-------------|--------------|
| Free | 10 | 1,000 |
| Starter | 60 | 50,000 |
| Professional | 300 | 500,000 |
| Enterprise | Unlimited | Unlimited |

---

## API VERSIONING

### Versioning Strategy

Use URL-based versioning:
```
/api/v1/...  (current stable)
/api/v2/...  (beta, new features)
```

### Version Lifecycle

1. **v1 (stable):** Production use for 18+ months
2. **v2 (beta):** New features, opt-in
3. **v1 (maintenance):** Security fixes only
4. **v1 (deprecated):** 6-month warning before removal

### Deprecation Header

```
Deprecated: true
Sunset: Wed, 01 Jan 2027 00:00:00 GMT
Link: </api/v2/strategies>; rel="successor-version"
```

---

## EXAMPLE ENDPOINTS

### Create Strategy

```
POST /api/v1/strategies

Request:
{
  "brand_id": "brand_123",
  "market_id": "enterprise-saas",
  "depth": "standard",
  "async": true
}

Response (202 Accepted - async):
{
  "success": true,
  "data": {
    "job_id": "job_456",
    "status": "queued"
  }
}

Response (200 OK - sync):
{
  "success": true,
  "data": {
    "id": "strat_789",
    "title": "Enterprise SaaS Strategy",
    "status": "draft",
    "market_opportunity": 85.5,
    "tactics": [...],
    "confidence_score": 0.87
  }
}
```

### List Strategies

```
GET /api/v1/strategies?status=approved&page=1&per_page=20

Response:
{
  "success": true,
  "data": [
    {
      "id": "strat_789",
      "title": "Enterprise SaaS Strategy",
      "status": "approved",
      "created_at": "2026-05-01T10:00:00Z"
    },
    ...
  ],
  "pagination": {
    "page": 1,
    "per_page": 20,
    "total": 150,
    "total_pages": 8
  }
}
```

### Get Strategy Details

```
GET /api/v1/strategies/strat_789

Response:
{
  "success": true,
  "data": {
    "id": "strat_789",
    "title": "Enterprise SaaS Strategy",
    "status": "approved",
    "executive_summary": "...",
    "market_opportunity": 85.5,
    "confidence_score": 0.87,
    "tactics": [
      {
        "name": "Sales team expansion",
        "timeline_weeks": 8,
        "resources": "high",
        "confidence": 0.92
      }
    ],
    "created_at": "2026-05-01T10:00:00Z",
    "approved_at": "2026-05-03T14:30:00Z"
  }
}
```

### Export Strategy to Presentation

```
POST /api/v1/strategies/strat_789/export

Request:
{
  "format": "pptx",
  "include_speaker_notes": true
}

Response (202 Accepted):
{
  "success": true,
  "data": {
    "job_id": "export_job_123",
    "status": "generating",
    "polling_url": "/api/v1/jobs/export_job_123"
  }
}
```

---

## HTTP STATUS CODES

| Code | Usage |
|------|-------|
| 200 OK | Successful GET, PUT, DELETE |
| 201 Created | Successful POST |
| 202 Accepted | Async job submitted |
| 204 No Content | Successful DELETE with no response body |
| 400 Bad Request | Malformed request |
| 401 Unauthorized | Missing/invalid auth |
| 403 Forbidden | Insufficient permissions |
| 404 Not Found | Resource not found |
| 409 Conflict | Resource conflict |
| 422 Unprocessable Entity | Validation failed |
| 429 Too Many Requests | Rate limited |
| 500 Internal Server Error | Server error |
| 503 Service Unavailable | External service down |

---

## REQUEST/RESPONSE EXAMPLES

### Strategy Creation Request

```json
POST /api/v1/strategies

{
  "brand_id": "brand_123",
  "market_id": "enterprise-saas-2026",
  "depth": "deep",
  "context": {
    "competitor_focus": "slack",
    "geographic_focus": "north-america"
  },
  "async": true
}
```

### Webhook Event

```json
POST https://customer.com/webhooks/strategy-complete

{
  "event_type": "strategy.complete",
  "event_id": "evt_123456",
  "organization_id": "org_789",
  "data": {
    "strategy_id": "strat_999",
    "title": "Q3 2026 Enterprise Strategy",
    "status": "ready_for_review"
  },
  "timestamp": "2026-05-06T12:00:00Z",
  "signature": "sha256=abc123..."
}
```

---

## DEPRECATION POLICY

### Notification Timeline

- **Month 1-3:** Announce deprecation, provide migration guide
- **Month 4-6:** Emit `Deprecated` header in responses
- **Month 7-9:** Reduce support level
- **Month 10-12:** Remove endpoint

### Migration Path

Old endpoint: `GET /api/v1/strategies/list`  
New endpoint: `GET /api/v1/strategies`  
Sunset date: 2026-11-01

---

**API Standards Owner:** Platform Engineering  
**Last Updated:** 2026-05-06  
**Next Review:** 2026-08-06

