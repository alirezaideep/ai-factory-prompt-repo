# Planning Service — Backend API Endpoints

## Base URL
```
/api/v1/planning/
```

## Endpoints

### POST /api/v1/planning/generate
Generate a plan from a task.

```json
// Request
{
  "task_id": "task_abc123",
  "goal": "Add urgent order support with priority processing",
  "scope": {
    "modules": ["order_management", "notifications"],
    "constraints": ["Must not break existing order flow", "Must support rollback"]
  },
  "priority": "high",
  "context_hints": ["This is a B2B glass manufacturing system"]
}

// Response 202 (Accepted - async processing)
{
  "success": true,
  "data": {
    "plan_id": "plan_xyz789",
    "status": "generating",
    "estimated_completion_seconds": 45
  }
}
```

### GET /api/v1/planning/plans/:planId
Get plan details.

```json
// Response 200
{
  "success": true,
  "data": {
    "plan_id": "plan_xyz789",
    "task_id": "task_abc123",
    "version": 1,
    "status": "generated",
    "dag": { ... },
    "cost_estimate": { ... },
    "risk_assessment": { ... },
    "timeline": { ... },
    "created_at": "2026-01-15T10:30:00Z"
  }
}
```

### POST /api/v1/planning/plans/:planId/regenerate
Regenerate plan with feedback from human.

```json
// Request
{
  "feedback": "Split step 3 into two smaller steps. Add a rollback step.",
  "modifications": [
    {"step_id": "step_3", "instruction": "Split into API implementation and validation separately"},
    {"add_step": {"after": "step_6", "description": "Add rollback migration script"}}
  ]
}

// Response 202
{
  "success": true,
  "data": {
    "plan_id": "plan_xyz789",
    "version": 2,
    "status": "regenerating",
    "estimated_completion_seconds": 30
  }
}
```

### GET /api/v1/planning/plans/:planId/versions
Get version history of a plan.

```json
// Response 200
{
  "success": true,
  "data": {
    "plan_id": "plan_xyz789",
    "versions": [
      {"version": 1, "status": "superseded", "created_at": "...", "feedback": null},
      {"version": 2, "status": "generated", "created_at": "...", "feedback": "Split step 3..."}
    ]
  }
}
```

### GET /api/v1/planning/plans/:planId/diff?from=1&to=2
Get diff between two plan versions.

```json
// Response 200
{
  "success": true,
  "data": {
    "from_version": 1,
    "to_version": 2,
    "changes": {
      "nodes_added": ["step_3b", "step_9"],
      "nodes_removed": [],
      "nodes_modified": ["step_3"],
      "edges_added": [{"from": "step_3", "to": "step_3b"}],
      "edges_removed": [],
      "cost_delta": {"tokens": +3000, "usd": +0.09},
      "timeline_delta": {"minutes": +2}
    }
  }
}
```

### POST /api/v1/planning/validate
Validate a plan schema (used internally and for testing).

```json
// Request
{
  "plan": { ... }  // Full plan schema
}

// Response 200
{
  "success": true,
  "data": {
    "valid": true,
    "warnings": ["Step 4 has no tests covering it"],
    "errors": []
  }
}
```

### GET /api/v1/planning/templates
Get plan templates for common task types.

```json
// Response 200
{
  "success": true,
  "data": {
    "templates": [
      {"id": "add_feature", "name": "Add New Feature", "steps": 8},
      {"id": "fix_bug", "name": "Fix Bug", "steps": 5},
      {"id": "refactor", "name": "Refactor Module", "steps": 6},
      {"id": "add_api", "name": "Add API Endpoint", "steps": 4}
    ]
  }
}
```

## Internal Events (Published to Message Queue)

| Event | Topic | Payload |
|-------|-------|---------|
| Plan generated | `planning.plan.generated` | `{plan_id, task_id, version}` |
| Plan regenerated | `planning.plan.regenerated` | `{plan_id, task_id, version, feedback}` |
| Plan validation failed | `planning.plan.validation_failed` | `{plan_id, errors}` |

## Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `PLAN_001` | 404 | Plan not found |
| `PLAN_002` | 409 | Plan already generating (wait) |
| `PLAN_003` | 422 | Invalid task scope (no modules) |
| `PLAN_004` | 500 | LLM generation failed |
| `PLAN_005` | 429 | Rate limit exceeded |
| `PLAN_006` | 422 | DAG validation failed (cycle detected) |
