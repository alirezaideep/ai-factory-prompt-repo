# Command UI — Backend API Endpoints (BFF Layer)

## Overview

The Command UI BFF (Backend-For-Frontend) aggregates data from multiple microservices into frontend-optimized responses. It runs as Next.js API routes.

## Base URL
```
/api/v1/
```

## Endpoints

### Tasks

#### POST /api/v1/tasks
Create a new task.
```json
// Request
{
  "title": "Add urgent order support",
  "description": "Users need to mark orders as urgent with priority processing",
  "goal": "Enable urgent order workflow with 2-hour SLA",
  "scope": {
    "modules": ["order_management", "notifications"],
    "constraints": ["Must not break existing order flow", "Must support rollback"]
  },
  "priority": "high",
  "attachments": ["file_id_1", "file_id_2"],
  "metadata": {
    "requested_by": "user_id",
    "project_id": "proj_abc"
  }
}

// Response 201
{
  "success": true,
  "data": {
    "id": "task_abc123",
    "status": "submitted",
    "created_at": "2026-01-15T10:30:00Z",
    "estimated_planning_time": "30s"
  }
}
```

#### GET /api/v1/tasks
List tasks with filtering and pagination.
```
Query params: ?status=planning&priority=high&page=1&limit=25&sort=-created_at
```

#### GET /api/v1/tasks/:id
Get task detail including plan, execution status, and history.

#### PATCH /api/v1/tasks/:id
Update task (only draft/submitted status).

#### DELETE /api/v1/tasks/:id
Cancel task (soft delete, only if not executing).

---

### Plans

#### GET /api/v1/plans/:taskId
Get generated plan for a task.
```json
// Response
{
  "success": true,
  "data": {
    "id": "plan_xyz",
    "task_id": "task_abc123",
    "version": 2,
    "status": "awaiting_approval",
    "dag": {
      "nodes": [...],
      "edges": [...]
    },
    "cost_estimate": {
      "tokens": 45000,
      "estimated_usd": 1.35,
      "compute_minutes": 8
    },
    "risk_assessment": {
      "score": 0.4,
      "factors": ["modifies shared module", "no existing tests"]
    },
    "timeline": {
      "estimated_minutes": 15,
      "steps": [...]
    }
  }
}
```

#### POST /api/v1/plans/:planId/approve
Approve a plan.
```json
// Request
{
  "decision": "approved",
  "comments": "Looks good, proceed",
  "modified_steps": []
}
```

#### POST /api/v1/plans/:planId/reject
Reject a plan with reason.
```json
// Request
{
  "decision": "rejected",
  "reason": "Scope too broad, split into two tasks",
  "suggestions": ["Split order and notification into separate tasks"]
}
```

#### POST /api/v1/plans/:planId/modify
Request plan modification.
```json
// Request
{
  "decision": "modify",
  "modifications": [
    {"step_id": "step_3", "change": "Add rollback mechanism"},
    {"step_id": "step_5", "change": "Include integration test"}
  ]
}
```

---

### Executions

#### GET /api/v1/executions
List executions with filtering.
```
Query params: ?status=running&agent=code_agent&page=1&limit=25
```

#### GET /api/v1/executions/:id
Get execution detail with step progress.
```json
// Response
{
  "success": true,
  "data": {
    "id": "exec_123",
    "plan_id": "plan_xyz",
    "status": "running",
    "progress": {
      "total_steps": 8,
      "completed": 5,
      "current_step": {
        "id": "step_6",
        "agent": "code_agent",
        "description": "Implement order priority field",
        "started_at": "2026-01-15T10:35:00Z",
        "logs_url": "/api/v1/executions/exec_123/logs/step_6"
      }
    },
    "cost": {
      "tokens_used": 32000,
      "usd_spent": 0.96
    }
  }
}
```

#### GET /api/v1/executions/:id/logs/:stepId
Stream execution logs (SSE).

#### POST /api/v1/executions/:id/retry
Retry failed execution from last failed step.

#### POST /api/v1/executions/:id/abort
Abort running execution.

---

### Repository

#### GET /api/v1/repo/tree
Get file tree of prompt repository.
```json
// Response
{
  "success": true,
  "data": {
    "tree": [
      {"path": "shared/backend.md", "type": "file", "size": 4520, "modified": "2026-01-14"},
      {"path": "features/", "type": "directory", "children": [...]}
    ]
  }
}
```

#### GET /api/v1/repo/file/:path
Get file content.
```json
// Response
{
  "success": true,
  "data": {
    "path": "shared/backend.md",
    "content": "# Backend Architecture\n...",
    "version": "v1.5.0",
    "last_modified": "2026-01-14T08:00:00Z",
    "modified_by": "team_lead_a"
  }
}
```

#### PUT /api/v1/repo/file/:path
Update file content (team lead only).
```json
// Request
{
  "content": "# Updated Backend Architecture\n...",
  "commit_message": "feat(backend): add rate limiting section",
  "version_bump": "patch"
}
```

#### GET /api/v1/repo/file/:path/history
Get version history of a file.

#### GET /api/v1/repo/diff/:path?from=v1.4.0&to=v1.5.0
Get diff between two versions.

---

### Feedback

#### POST /api/v1/feedback
Submit feedback on execution output.
```json
// Request
{
  "execution_id": "exec_123",
  "step_id": "step_6",
  "rating": 4,
  "decision": "accepted_with_edits",
  "edits": {
    "original_lines": [15, 16, 17],
    "modified_content": "..."
  },
  "comment": "Logic correct but naming inconsistent"
}
```

#### GET /api/v1/feedback/analytics
Get feedback analytics (success rates, edit rates).

---

### WebSocket Events

#### Connection
```
ws://factory.internal/ws?token=jwt_token
```

#### Events (Server → Client)
| Event | Payload | Trigger |
|-------|---------|---------|
| `task.status_changed` | `{task_id, old_status, new_status}` | Any task status change |
| `plan.generated` | `{task_id, plan_id, summary}` | Planning complete |
| `approval.requested` | `{plan_id, priority, deadline}` | Plan needs approval |
| `execution.step.started` | `{exec_id, step_id, agent}` | Agent starts step |
| `execution.step.completed` | `{exec_id, step_id, result}` | Agent completes step |
| `execution.failed` | `{exec_id, step_id, error}` | Execution error |
| `execution.completed` | `{exec_id, summary}` | All steps done |
| `repo.updated` | `{path, version, author}` | File merged |
