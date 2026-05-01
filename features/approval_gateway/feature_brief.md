# Approval Gateway — Feature Brief

## Module Identity

| Field | Value |
|-------|-------|
| Module ID | `approval_gateway` |
| Layer | 3 (Human Control) |
| Owner | Backend Team Lead |
| Status | Planned |
| Priority | Critical (safety layer) |

## Purpose

The Approval Gateway is the **safety layer** between AI planning and AI execution. No plan executes without explicit human approval. It enforces governance policies, manages approval queues, tracks SLAs, and provides audit trails for all decisions.

## Core Capabilities

### 1. Approval Queue Management
- Priority-sorted queue of pending approvals
- Assignment to appropriate reviewer (based on module ownership)
- SLA tracking (time-to-review alerts)
- Batch approval for low-risk items (configurable)
- Delegation and escalation rules

### 2. Decision Capture
- Three decision types: Approve, Modify, Reject
- Mandatory reason for Reject
- Optional comments for Approve
- Structured modification requests for Modify
- Decision audit trail (who, when, why)

### 3. Policy Enforcement
- Cost threshold policies (auto-escalate if > $X)
- Risk threshold policies (require senior approval if risk > 0.7)
- Scope policies (require PM approval if touches > 3 modules)
- Time policies (no auto-approve outside business hours)
- Blacklist policies (never auto-approve certain modules)

### 4. Auto-Approval (Optional, Feature-Flagged)
- Configurable rules for automatic approval
- Only for low-risk, low-cost, single-module changes
- Requires explicit opt-in by Project Owner
- Full audit trail even for auto-approved items
- Can be disabled globally at any time

### 5. Notification & Escalation
- Instant notification to reviewer when plan is ready
- Reminder after 50% of SLA elapsed
- Escalation to backup reviewer at 80% SLA
- Escalation to Project Owner at 100% SLA
- Channels: in-app, Slack, email (configurable)

## API Endpoints

### POST /api/v1/approvals/submit
Submit a plan for approval.
```json
{
  "plan_id": "plan_xyz",
  "task_id": "task_abc",
  "submitted_by": "planning_service",
  "priority": "high",
  "cost_estimate_usd": 1.35,
  "risk_score": 0.4,
  "affected_modules": ["order_management", "notifications"],
  "sla_minutes": 60
}
```

### GET /api/v1/approvals/queue
Get pending approvals for current user.
```json
{
  "items": [
    {
      "id": "appr_123",
      "plan_id": "plan_xyz",
      "task_title": "Add urgent orders",
      "priority": "high",
      "risk": "medium",
      "cost_usd": 1.35,
      "submitted_at": "2026-01-15T10:30:00Z",
      "sla_deadline": "2026-01-15T11:30:00Z",
      "sla_remaining_minutes": 45
    }
  ]
}
```

### POST /api/v1/approvals/:id/decide
Record approval decision.
```json
{
  "decision": "approved",  // approved | modified | rejected
  "reason": "Looks good",
  "modifications": [],  // Only for "modified"
  "reviewer_id": "user_abc"
}
```

### GET /api/v1/approvals/:id/history
Get decision history for an approval.

### GET /api/v1/approvals/policies
Get active approval policies.

### PUT /api/v1/approvals/policies
Update approval policies (admin only).

## Data Model

### Table: approvals
```sql
CREATE TABLE approvals (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  plan_id UUID NOT NULL REFERENCES plans(id),
  task_id UUID NOT NULL REFERENCES tasks(id),
  
  status VARCHAR(20) NOT NULL DEFAULT 'pending',
  -- Status: pending | approved | modified | rejected | escalated | auto_approved
  
  priority VARCHAR(10) NOT NULL,
  risk_score DECIMAL(3,2) NOT NULL,
  cost_estimate_usd DECIMAL(10,2) NOT NULL,
  affected_modules TEXT[] NOT NULL,
  
  assigned_to UUID REFERENCES users(id),
  decided_by UUID REFERENCES users(id),
  decision_reason TEXT,
  modifications JSONB,
  
  sla_deadline TIMESTAMP WITH TIME ZONE NOT NULL,
  escalated_at TIMESTAMP WITH TIME ZONE,
  decided_at TIMESTAMP WITH TIME ZONE,
  
  auto_approved BOOLEAN NOT NULL DEFAULT FALSE,
  policy_applied VARCHAR(50),
  
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);
```

### Table: approval_policies
```sql
CREATE TABLE approval_policies (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(100) NOT NULL,
  description TEXT,
  
  conditions JSONB NOT NULL,
  -- {"max_cost_usd": 0.50, "max_risk": 0.2, "max_modules": 1}
  
  action VARCHAR(20) NOT NULL,
  -- Action: auto_approve | require_senior | require_owner | block
  
  enabled BOOLEAN NOT NULL DEFAULT TRUE,
  priority INTEGER NOT NULL DEFAULT 0,  -- Higher = evaluated first
  
  created_by UUID REFERENCES users(id),
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);
```

## Workflow

```
Plan Generated
    ↓
[Check Policies] → Does any policy trigger auto-approve?
    ↓ No                    ↓ Yes (and FF enabled)
[Assign Reviewer]       [Auto-Approve]
    ↓                       ↓
[Notify Reviewer]       [Log + Proceed]
    ↓
[Wait for Decision]
    ↓
[SLA Monitor] → 50% elapsed? → Reminder
              → 80% elapsed? → Escalate to backup
              → 100% elapsed? → Escalate to Owner
    ↓
[Decision Received]
    ↓
[Record in Audit Trail]
    ↓
[Route Decision]
    ├── Approved → Notify Orchestrator
    ├── Modified → Notify Planning Service (iterate)
    └── Rejected → Notify Planning Service (iterate with feedback)
```

## Success Metrics

| Metric | Target |
|--------|--------|
| Average review time | < 30 minutes |
| SLA compliance | > 95% |
| Decision reversal rate | < 5% |
| Auto-approval accuracy | > 99% (no false approvals) |
