# Orchestrator — API Endpoints

## Base Path: /api/v1/orchestrator

---

## POST /api/v1/orchestrator/executions
Start a new plan execution.

### Request
```json
{
  "plan_id": "uuid",
  "project_id": "uuid",
  "config": {
    "max_parallel": 3,
    "retry_policy": {
      "max_retries": 2,
      "backoff_seconds": [30, 120]
    },
    "timeout_minutes": 120,
    "budget_limit_usd": 25.00,
    "pause_on_failure": true,
    "require_approval_per_step": false
  }
}
```

### Response (201)
```json
{
  "execution_id": "uuid",
  "status": "pending_approval",
  "dag": {
    "nodes": [...],
    "edges": [...]
  },
  "estimated_cost_usd": 12.50,
  "estimated_duration_minutes": 45
}
```

---

## GET /api/v1/orchestrator/executions/:id
Get execution status and progress.

### Response (200)
```json
{
  "id": "uuid",
  "plan_id": "uuid",
  "status": "running",
  "progress": {
    "total_steps": 8,
    "completed": 3,
    "running": 2,
    "pending": 3,
    "failed": 0
  },
  "steps": [
    {
      "id": "step_1",
      "name": "Update data model",
      "agent_type": "code_agent",
      "status": "completed",
      "started_at": "2025-01-15T10:00:00Z",
      "completed_at": "2025-01-15T10:02:30Z",
      "tokens_used": 4200,
      "cost_usd": 0.13
    },
    {
      "id": "step_2",
      "name": "Update API endpoints",
      "agent_type": "code_agent",
      "status": "running",
      "started_at": "2025-01-15T10:02:35Z",
      "progress_percent": 60
    }
  ],
  "cost_so_far_usd": 2.30,
  "elapsed_minutes": 12
}
```

---

## POST /api/v1/orchestrator/executions/:id/pause
Pause a running execution (completes current steps, doesn't start new ones).

### Response (200)
```json
{
  "status": "paused",
  "running_steps": ["step_3"],
  "message": "Execution paused. Step 3 will complete but no new steps will start."
}
```

---

## POST /api/v1/orchestrator/executions/:id/resume
Resume a paused execution.

### Response (200)
```json
{
  "status": "running",
  "next_steps": ["step_4", "step_5"]
}
```

---

## POST /api/v1/orchestrator/executions/:id/cancel
Cancel execution (attempts graceful stop of running steps).

### Response (200)
```json
{
  "status": "cancelled",
  "completed_steps": 3,
  "cancelled_steps": 5,
  "cost_incurred_usd": 2.30
}
```

---

## POST /api/v1/orchestrator/executions/:id/steps/:stepId/retry
Retry a failed step with optional modifications.

### Request
```json
{
  "modifications": {
    "additional_context": "Previous attempt failed because...",
    "model_override": "claude-3-opus",
    "timeout_override_seconds": 600
  }
}
```

### Response (200)
```json
{
  "step_id": "step_3",
  "status": "retrying",
  "attempt": 2,
  "max_attempts": 3
}
```

---

## POST /api/v1/orchestrator/executions/:id/steps/:stepId/skip
Skip a step (mark as skipped, allow dependents to proceed).

### Request
```json
{
  "reason": "Manual implementation preferred for this step",
  "mock_output": {
    "files": [],
    "summary": "Skipped - will be implemented manually"
  }
}
```

---

## GET /api/v1/orchestrator/executions/:id/dag
Get the execution DAG visualization data.

### Response (200)
```json
{
  "nodes": [
    {"id": "step_1", "label": "Update data model", "status": "completed", "agent": "code_agent"},
    {"id": "step_2", "label": "Update API", "status": "running", "agent": "code_agent"},
    {"id": "step_3", "label": "Update frontend", "status": "pending", "agent": "code_agent"},
    {"id": "step_4", "label": "Write tests", "status": "pending", "agent": "test_agent"},
    {"id": "step_5", "label": "Code review", "status": "pending", "agent": "review_agent"}
  ],
  "edges": [
    {"from": "step_1", "to": "step_2"},
    {"from": "step_1", "to": "step_3"},
    {"from": "step_2", "to": "step_4"},
    {"from": "step_3", "to": "step_4"},
    {"from": "step_4", "to": "step_5"}
  ]
}
```

---

## GET /api/v1/orchestrator/executions/:id/logs
Stream execution logs (SSE).

### Response (200, text/event-stream)
```
event: step_started
data: {"step_id": "step_2", "agent_type": "code_agent", "timestamp": "..."}

event: tool_call
data: {"step_id": "step_2", "tool": "read_file", "input": {"path": "..."}, "timestamp": "..."}

event: step_completed
data: {"step_id": "step_2", "duration_ms": 45000, "tokens": 8500, "timestamp": "..."}

event: step_failed
data: {"step_id": "step_3", "error": "Compilation error", "attempt": 1, "timestamp": "..."}
```

---

## WebSocket: /ws/orchestrator/executions/:id
Real-time execution updates.

### Messages (Server → Client)
```json
{"type": "progress", "data": {"step_id": "step_2", "percent": 75}}
{"type": "step_completed", "data": {"step_id": "step_2", "output_summary": "..."}}
{"type": "step_failed", "data": {"step_id": "step_3", "error": "...", "recoverable": true}}
{"type": "approval_needed", "data": {"step_id": "step_4", "reason": "High-risk operation"}}
{"type": "execution_completed", "data": {"total_cost": 5.20, "total_time_minutes": 32}}
```

### Messages (Client → Server)
```json
{"type": "approve_step", "step_id": "step_4"}
{"type": "skip_step", "step_id": "step_4", "reason": "..."}
{"type": "pause"}
{"type": "resume"}
{"type": "cancel"}
```
