# Approval Gateway — API Endpoints

## Base Path: `/api/v1/approvals`

### Approval Requests

#### GET /api/v1/approvals
List approval requests with filtering.

**Query Params:** `?status=pending&project_id=xxx&type=plan_review&limit=20`

**Response:**
```json
{
  "data": [
    {
      "id": "apr-uuid",
      "type": "plan_review",
      "status": "pending",
      "projectId": "proj-uuid",
      "executionId": "exec-uuid",
      "requestedBy": "planning-service",
      "assignedTo": "user-uuid",
      "title": "Review execution plan: Add user authentication",
      "summary": "12 steps, estimated 45 min, $3.20 budget",
      "priority": "normal",
      "createdAt": "2025-02-20T10:30:00Z",
      "expiresAt": "2025-02-21T10:30:00Z",
      "context": {
        "planSchema": { ... },
        "riskAssessment": { "level": "medium", "factors": [...] }
      }
    }
  ],
  "meta": { "total": 5, "pending": 3 }
}
```

#### GET /api/v1/approvals/{id}
Get full approval request details with context.

#### POST /api/v1/approvals/{id}/decide
Submit approval decision.

**Request:**
```json
{
  "decision": "approve",
  "comment": "Looks good, proceed with execution",
  "modifications": null
}
```

Alternative decisions:
```json
{
  "decision": "modify",
  "comment": "Split step 5 into two smaller steps",
  "modifications": {
    "steps": [
      { "action": "split", "stepId": "step-5", "into": ["step-5a", "step-5b"] }
    ]
  }
}
```

```json
{
  "decision": "reject",
  "comment": "Scope too large, break into smaller PRs",
  "reason": "scope_too_large"
}
```

#### POST /api/v1/approvals/{id}/delegate
Delegate approval to another team member.

**Request:**
```json
{
  "delegateTo": "user-uuid",
  "reason": "Domain expert needed for database changes"
}
```

### Approval Policies

#### GET /api/v1/approvals/policies
List approval policies (rules for auto-routing).

#### POST /api/v1/approvals/policies
Create a new approval policy.

**Request:**
```json
{
  "name": "Auto-approve documentation",
  "conditions": {
    "type": "plan_review",
    "riskLevel": "low",
    "stepTypes": ["documentation"],
    "estimatedCost": { "max": 1.00 }
  },
  "action": "auto_approve",
  "notifyOnly": ["team-lead-uuid"]
}
```

### Escalation

#### POST /api/v1/approvals/{id}/escalate
Escalate a timed-out or complex approval.

**Request:**
```json
{
  "escalateTo": "admin-uuid",
  "reason": "No response within 24 hours"
}
```
