# Backend Architecture

## Overview

The AI Software Factory backend consists of multiple microservices communicating via REST APIs and event-driven messaging. Each service is independently deployable, versioned, and testable.

## Tech Stack

| Layer | Technology | Version |
|-------|-----------|---------|
| Language | Python 3.12+ | Primary for AI services |
| Language | TypeScript 5.x | For Command UI BFF |
| Framework | FastAPI | All Python services |
| Framework | Next.js API Routes | BFF layer |
| ORM | SQLAlchemy 2.0 + Alembic | Database access |
| Queue | Redis Streams + Bull | Job processing |
| Cache | Redis 7.x | Session, rate limiting |
| Database | PostgreSQL 16 + pgvector | Primary store |
| Analytics DB | ClickHouse | Time-series, metrics |
| Vector Store | ChromaDB / pgvector | Embeddings |
| LLM Gateway | LangChain + LiteLLM | Multi-provider abstraction |
| Container | Docker | Service isolation |
| Orchestration | Kubernetes 1.29+ | Production deployment |

## Service Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      API Gateway (Kong/Traefik)               │
├─────────────────────────────────────────────────────────────┤
│  Command UI BFF  │  Planning API  │  Prompt Factory API      │
│  (Next.js)       │  (FastAPI)     │  (FastAPI)               │
├─────────────────────────────────────────────────────────────┤
│  Orchestrator    │  Agent Runtime │  Knowledge Base API       │
│  (LangGraph)     │  (FastAPI)     │  (FastAPI)               │
├─────────────────────────────────────────────────────────────┤
│  Feedback Engine │  Auth Service  │  Notification Service     │
│  (FastAPI)       │  (FastAPI)     │  (FastAPI)               │
├─────────────────────────────────────────────────────────────┤
│         PostgreSQL  │  Redis  │  ClickHouse  │  ChromaDB     │
└─────────────────────────────────────────────────────────────┘
```

## API Standards

### Endpoint Naming
- RESTful: `/{version}/{resource}/{id?}/{sub-resource?}`
- Example: `/v1/tasks/abc123/plans`
- Versioned: All APIs prefixed with `/v1/`

### Request/Response Format
```json
// Success Response
{
  "success": true,
  "data": { ... },
  "meta": {
    "timestamp": "2026-01-15T10:30:00Z",
    "request_id": "req_abc123",
    "version": "v1"
  }
}

// Error Response
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Human-readable message",
    "details": [ ... ],
    "request_id": "req_abc123"
  }
}
```

### Authentication
- JWT tokens (RS256) for service-to-service
- OAuth 2.0 + OIDC for user authentication
- API keys for external integrations
- RBAC with role hierarchy: `admin > team_lead > developer > viewer`

### Rate Limiting
- Per-user: 100 req/min (standard), 1000 req/min (premium)
- Per-service: 10,000 req/min
- LLM calls: Token-based budgeting per project

## Database Conventions

### Naming
- Tables: `snake_case`, plural (e.g., `tasks`, `plan_schemas`)
- Columns: `snake_case` (e.g., `created_at`, `plan_id`)
- Indexes: `idx_{table}_{column}` (e.g., `idx_tasks_status`)
- Foreign keys: `fk_{table}_{ref_table}` (e.g., `fk_tasks_projects`)

### Standard Columns (every table)
```sql
id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
created_by  UUID REFERENCES users(id),
version     INTEGER NOT NULL DEFAULT 1,
is_deleted  BOOLEAN NOT NULL DEFAULT FALSE
```

### Migration Strategy
- Alembic for schema migrations
- Forward-only migrations (no down migrations in production)
- Migration naming: `{timestamp}_{description}.py`
- Every schema change requires a changelog entry

## Service Communication

### Synchronous (REST)
- Service-to-service via internal DNS (Kubernetes)
- Circuit breaker pattern (tenacity + custom middleware)
- Timeout: 30s default, 120s for LLM calls
- Retry: 3 attempts with exponential backoff

### Asynchronous (Events)
- Redis Streams for event bus
- Event naming: `{service}.{entity}.{action}` (e.g., `planning.plan.created`)
- Dead letter queue for failed events
- At-least-once delivery guarantee

### Event Schema
```json
{
  "event_id": "evt_abc123",
  "event_type": "planning.plan.created",
  "timestamp": "2026-01-15T10:30:00Z",
  "source": "planning-service",
  "data": { ... },
  "metadata": {
    "correlation_id": "corr_xyz",
    "causation_id": "evt_prev123"
  }
}
```

## Error Handling

### Error Codes
| Code | HTTP Status | Meaning |
|------|-------------|---------|
| VALIDATION_ERROR | 400 | Invalid input |
| UNAUTHORIZED | 401 | Missing/invalid auth |
| FORBIDDEN | 403 | Insufficient permissions |
| NOT_FOUND | 404 | Resource not found |
| CONFLICT | 409 | State conflict |
| RATE_LIMITED | 429 | Too many requests |
| LLM_ERROR | 502 | LLM provider failure |
| TIMEOUT | 504 | Operation timed out |

### Retry Strategy
- Idempotent operations: auto-retry with exponential backoff
- Non-idempotent: fail fast, notify user
- LLM calls: retry with fallback provider (Claude → GPT-4 → local)

## Testing Strategy

| Level | Tool | Coverage Target |
|-------|------|----------------|
| Unit | pytest | >80% |
| Integration | pytest + testcontainers | Critical paths |
| Contract | Pact | All service boundaries |
| E2E | Playwright | Happy paths |
| Load | Locust | P95 < 500ms |

## Logging & Observability

- Structured JSON logging (structlog)
- Correlation ID propagated across services
- OpenTelemetry for distributed tracing
- Prometheus metrics exposed at `/metrics`
- Log levels: DEBUG (dev), INFO (staging), WARNING (prod)
