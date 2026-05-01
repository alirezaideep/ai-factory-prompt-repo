# Command UI — Data Model

## Overview

The Command UI BFF does not own its own database. It aggregates data from other services. However, it maintains a local cache (Redis) for session state and UI preferences.

## Cached Entities

### UserSession (Redis)
```
Key: session:{user_id}
TTL: 8 hours
```
```json
{
  "user_id": "usr_abc",
  "role": "team_lead",
  "project_id": "proj_xyz",
  "preferences": {
    "theme": "dark",
    "sidebar_collapsed": false,
    "default_view": "executions",
    "notifications_enabled": true
  },
  "last_active": "2026-01-15T10:30:00Z"
}
```

### NotificationQueue (Redis Stream)
```
Stream: notifications:{user_id}
```
```json
{
  "id": "notif_123",
  "type": "approval_requested",
  "title": "Plan ready for review",
  "body": "Task 'Add urgent orders' has a plan ready",
  "link": "/approvals/plan_xyz",
  "priority": "high",
  "read": false,
  "created_at": "2026-01-15T10:30:00Z"
}
```

## Aggregated Views (from other services)

### Dashboard Summary
Aggregated from: Planning Service + Orchestrator + Feedback Engine
```json
{
  "active_tasks": 12,
  "pending_approvals": 3,
  "running_executions": 5,
  "completed_today": 8,
  "success_rate_7d": 0.87,
  "total_cost_today_usd": 45.20,
  "agents_active": ["code_agent", "test_agent", "review_agent"]
}
```

### Task List View
Aggregated from: Planning Service
```json
{
  "id": "task_abc",
  "title": "Add urgent order support",
  "status": "executing",
  "priority": "high",
  "submitted_by": "Developer A",
  "submitted_at": "2026-01-15T09:00:00Z",
  "plan_version": 2,
  "progress_percent": 62,
  "cost_so_far_usd": 0.96
}
```

## UI State (LocalStorage)

```json
{
  "recent_files": ["shared/backend.md", "features/orders/back/api.md"],
  "pinned_executions": ["exec_123", "exec_456"],
  "filter_presets": {
    "my_tasks": {"submitted_by": "me", "status": ["submitted", "planning", "executing"]},
    "urgent": {"priority": "critical", "status": ["pending_approval"]}
  },
  "editor_settings": {
    "font_size": 14,
    "word_wrap": true,
    "show_line_numbers": true
  }
}
```
