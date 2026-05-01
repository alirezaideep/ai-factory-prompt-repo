# API Conventions

## URL Structure
```
/api/v{version}/{resource}
/api/v{version}/{resource}/:id
/api/v{version}/{resource}/:id/{sub-resource}
```

## Naming Rules
- Resources: plural nouns, kebab-case (`/api/v1/approval-requests`)
- Query params: camelCase (`?pageSize=20&sortBy=createdAt`)
- Request/response bodies: camelCase
- Database columns: snake_case
- Enum values: UPPER_SNAKE_CASE in DB, camelCase in API

## HTTP Methods
| Method | Purpose | Idempotent | Response |
|--------|---------|-----------|----------|
| GET | Read | Yes | 200 + body |
| POST | Create | No | 201 + body |
| PUT | Full update | Yes | 200 + body |
| PATCH | Partial update | Yes | 200 + body |
| DELETE | Remove | Yes | 204 no body |

## Pagination
```json
{
  "data": [...],
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "totalItems": 150,
    "totalPages": 8,
    "hasNext": true,
    "hasPrev": false
  }
}
```

## Filtering
```
GET /api/v1/orders?status=in_production&priority=high&createdAfter=2025-01-01
```

## Sorting
```
GET /api/v1/orders?sortBy=createdAt&sortOrder=desc
```

## Error Response Format
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Human-readable message",
    "details": [
      {"field": "email", "message": "Invalid email format"}
    ],
    "requestId": "uuid",
    "timestamp": "2025-01-15T10:00:00Z"
  }
}
```

## Error Codes
| HTTP Status | Code | Use Case |
|-------------|------|----------|
| 400 | VALIDATION_ERROR | Invalid input |
| 401 | UNAUTHORIZED | Missing/invalid auth |
| 403 | FORBIDDEN | Insufficient permissions |
| 404 | NOT_FOUND | Resource doesn't exist |
| 409 | CONFLICT | State conflict (e.g., already approved) |
| 422 | UNPROCESSABLE | Valid syntax but semantic error |
| 429 | RATE_LIMITED | Too many requests |
| 500 | INTERNAL_ERROR | Unexpected server error |

## Versioning
- URL-based: `/api/v1/`, `/api/v2/`
- Breaking changes require new version
- Non-breaking additions (new optional fields) allowed in same version
- Deprecation: minimum 3 months notice, `Sunset` header

## Authentication
- Bearer token in `Authorization` header
- Session cookie for browser clients
- API key for service-to-service
- All endpoints require auth unless explicitly marked `@public`

## Rate Limiting
- Default: 100 requests/minute per user
- Headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`
- Burst: 20 requests/second max

## Request Validation
- Use Zod schemas for all input validation
- Validate at controller level before service call
- Return all validation errors at once (not first-only)
- Sanitize all string inputs (trim, escape HTML)
