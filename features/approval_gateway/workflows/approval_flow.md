# Approval Gateway — Approval Flow Workflow

## Overview

Every significant action in the AI Factory requires human approval before execution. The Approval Gateway ensures humans remain in control while minimizing friction for low-risk operations.

## Flow Diagram

```
[Action Requested]
    ↓
[Policy Engine evaluates]
    ├── Auto-approve conditions met? → [Execute immediately]
    └── Requires human review →
        ↓
[Create Approval Request]
    ↓
[Assign to Team Lead / Approver]
    ↓
[Notify via UI + Email/Slack]
    ↓
┌─────────────────────────────────────────────┐
│         ITERATIVE REVIEW LOOP               │
│                                             │
│  [Approver reviews payload]                 │
│       ↓                                     │
│  [Decision]                                 │
│       ├── Approve → Exit loop → Execute     │
│       ├── Reject → Exit loop → Notify       │
│       └── Modify → Send back to originator  │
│                    → Originator adjusts      │
│                    → Resubmit               │
│                    → Loop                    │
│                                             │
│  [Timeout check]                            │
│       ├── Not expired → Wait                │
│       └── Expired → Escalate to next level  │
│                                             │
└─────────────────────────────────────────────┘
    ↓
[Decision recorded with reason]
    ↓
[Emit event: approval.decided]
    ↓
[Downstream system acts on decision]
```

## Approval Types and Their Payloads

### 1. Plan Approval
```json
{
  "type": "plan_approval",
  "payload": {
    "plan_id": "uuid",
    "goal": "Add priority field to order management",
    "total_steps": 8,
    "agents_involved": ["code_agent", "test_agent", "review_agent"],
    "estimated_cost_usd": 3.50,
    "estimated_duration_minutes": 45,
    "affected_modules": ["order_management"],
    "risk_assessment": "medium",
    "plan_summary": "1. Update data model → 2. Update API → 3. Update frontend → 4. Tests"
  }
}
```

### 2. Merge Request (Prompt Repo)
```json
{
  "type": "merge_request",
  "payload": {
    "branch": "feature/add-priority",
    "target": "main",
    "files_changed": 4,
    "changes": [
      {"path": "features/orders/back/data_model.md", "operation": "modified", "diff_summary": "Added priority column"},
      {"path": "features/orders/back/api_endpoints.md", "operation": "modified", "diff_summary": "Added priority filter"}
    ],
    "version_bump": "0.3.0 → 0.4.0"
  }
}
```

### 3. Execution Start
```json
{
  "type": "execution_start",
  "payload": {
    "plan_id": "uuid",
    "step_id": "uuid",
    "agent_type": "code_agent",
    "model": "claude-3-opus",
    "estimated_tokens": 25000,
    "estimated_cost_usd": 0.75,
    "files_to_modify": ["server/routes/orders.ts", "server/models/order.ts"],
    "sandbox_config": {"memory": "1Gi", "cpu": "2", "timeout": "300s"}
  }
}
```

### 4. Budget Override
```json
{
  "type": "budget_override",
  "payload": {
    "project_id": "uuid",
    "current_budget_usd": 50.00,
    "spent_usd": 48.50,
    "requested_increase_usd": 25.00,
    "reason": "Additional complexity discovered in payment module",
    "remaining_steps": 12
  }
}
```

## Auto-Approve Rules (Default)

| Condition | Auto-Approve |
|-----------|-------------|
| Risk = low AND cost < $1.00 | Yes |
| Type = merge_request AND files_changed <= 2 AND module not critical | Yes |
| Type = execution_start AND agent = test_agent | Yes |
| Type = execution_start AND agent = docs_agent | Yes |
| Retry of previously approved step (same scope) | Yes |
| Everything else | No |

## Escalation Matrix

| Timeout | Action |
|---------|--------|
| 30 min (low risk) | Reminder notification |
| 60 min (medium risk) | Escalate to project owner |
| 15 min (high risk) | Immediate escalation to CTO |
| 5 min (critical) | Immediate escalation + SMS |

## UI Integration

### Approval Queue (Command UI)
- Badge count on sidebar showing pending approvals
- Sorted by: risk level (desc) → created_at (asc)
- Quick actions: Approve / Reject / View Details
- Inline diff viewer for merge requests
- Cost/risk summary card for execution approvals

### Approval Detail View
- Full payload visualization
- Impact analysis (what modules/files affected)
- Cost breakdown
- Comment thread
- Modification history (if iterative)
- One-click approve/reject with required reason for reject

## Notification Channels

```python
NOTIFICATION_CONFIG = {
    "low": ["in_app"],
    "medium": ["in_app", "email"],
    "high": ["in_app", "email", "slack"],
    "critical": ["in_app", "email", "slack", "sms"]
}
```
