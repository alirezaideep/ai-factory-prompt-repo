# Approval Gateway — Data Model

## Table: approval_requests
```sql
CREATE TABLE approval_requests (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID NOT NULL REFERENCES projects(id),
  
  type VARCHAR(30) NOT NULL,
  -- Types: plan_approval | merge_request | execution_start | budget_override | 
  --        human_escalation | config_change
  
  title VARCHAR(200) NOT NULL,
  description TEXT,
  
  status VARCHAR(20) NOT NULL DEFAULT 'pending',
  -- Status: pending | approved | rejected | modified | expired
  
  payload JSONB NOT NULL,
  -- Content to approve (plan schema, merge diff, execution config, etc.)
  
  risk_level VARCHAR(10) NOT NULL DEFAULT 'medium',
  -- Risk: low | medium | high | critical
  
  estimated_cost_usd DECIMAL(10,4),
  estimated_tokens INTEGER,
  estimated_duration_minutes INTEGER,
  
  scope_summary TEXT,
  -- Human-readable summary of what will happen if approved
  
  requested_by VARCHAR(30) NOT NULL,
  -- Who/what requested: planning_service | orchestrator | prompt_factory | user
  
  assigned_to UUID REFERENCES users(id),
  -- Team lead or specific approver
  
  decision VARCHAR(20),
  decision_reason TEXT,
  modifications JSONB,
  -- If status = 'modified', what was changed before approval
  
  decided_by UUID REFERENCES users(id),
  
  expires_at TIMESTAMP WITH TIME ZONE,
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  decided_at TIMESTAMP WITH TIME ZONE
);

CREATE INDEX idx_approvals_status ON approval_requests(status);
CREATE INDEX idx_approvals_project ON approval_requests(project_id);
CREATE INDEX idx_approvals_assigned ON approval_requests(assigned_to);
CREATE INDEX idx_approvals_type ON approval_requests(type);
```

## Table: approval_policies
```sql
CREATE TABLE approval_policies (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID REFERENCES projects(id),  -- NULL = global policy
  
  name VARCHAR(100) NOT NULL,
  description TEXT,
  
  trigger_conditions JSONB NOT NULL,
  -- When this policy applies:
  -- {"type": "plan_approval", "risk_level": ["high", "critical"]}
  -- {"type": "execution_start", "estimated_cost_gt": 10.0}
  -- {"type": "merge_request", "modules": ["payments", "auth"]}
  
  required_approvers INTEGER NOT NULL DEFAULT 1,
  auto_approve_conditions JSONB,
  -- Conditions for auto-approval:
  -- {"risk_level": "low", "estimated_cost_lt": 1.0, "module_not_in": ["payments"]}
  
  escalation_timeout_minutes INTEGER DEFAULT 60,
  escalation_to UUID REFERENCES users(id),
  
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);
```

## Table: approval_comments
```sql
CREATE TABLE approval_comments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  approval_id UUID NOT NULL REFERENCES approval_requests(id),
  
  author_id UUID NOT NULL REFERENCES users(id),
  content TEXT NOT NULL,
  
  type VARCHAR(20) NOT NULL DEFAULT 'comment',
  -- Types: comment | question | suggestion | condition
  
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_approval_comments_approval ON approval_comments(approval_id);
```

## Approval Flow Logic

```python
class ApprovalEngine:
    """
    Determines whether approval is needed and routes to correct approver.
    """
    
    async def request_approval(
        self,
        project_id: str,
        type: str,
        payload: dict,
        requested_by: str
    ) -> ApprovalRequest:
        """
        Create approval request or auto-approve based on policies.
        """
        # Get applicable policies
        policies = await db.get_active_policies(project_id, type)
        
        # Check auto-approve conditions
        for policy in policies:
            if self._matches_auto_approve(policy, payload):
                # Auto-approve
                return await self._create_auto_approved(project_id, type, payload)
        
        # Determine risk level
        risk = self._assess_risk(type, payload)
        
        # Determine assignee
        assignee = await self._determine_assignee(project_id, type, risk)
        
        # Create pending request
        request = await db.create_approval_request(
            project_id=project_id,
            type=type,
            payload=payload,
            risk_level=risk,
            assigned_to=assignee,
            requested_by=requested_by,
            expires_at=now() + timedelta(minutes=self._get_timeout(risk))
        )
        
        # Notify assignee
        await notify(assignee, "New approval request", request)
        
        return request
    
    def _assess_risk(self, type: str, payload: dict) -> str:
        """
        Assess risk level based on type and content.
        """
        if type == "execution_start":
            cost = payload.get("estimated_cost_usd", 0)
            if cost > 50: return "critical"
            if cost > 10: return "high"
            if cost > 2: return "medium"
            return "low"
        
        if type == "merge_request":
            modules = payload.get("affected_modules", [])
            critical_modules = {"payments", "auth", "user_management"}
            if set(modules) & critical_modules:
                return "high"
            return "medium"
        
        if type == "plan_approval":
            steps = payload.get("total_steps", 0)
            if steps > 20: return "high"
            if steps > 10: return "medium"
            return "low"
        
        return "medium"
```

## Iterative Approval (Back-and-Forth)

```python
async def handle_modification(approval_id: str, modifications: dict, modifier_id: str):
    """
    When approver modifies the request (not just approve/reject).
    This triggers a back-and-forth loop:
    
    1. Approver modifies payload
    2. System updates the request
    3. Originator (Planning Service) receives modification
    4. Originator adjusts and resubmits
    5. Approver reviews again
    6. Loop until approved or rejected
    """
    request = await db.get_approval(approval_id)
    
    # Record modification
    await db.update_approval(approval_id, {
        "status": "modified",
        "modifications": modifications,
        "decided_by": modifier_id,
        "decided_at": now()
    })
    
    # Create comment explaining modification
    await db.create_approval_comment(approval_id, modifier_id, 
        f"Modified: {modifications.get('reason', 'See changes')}", type="suggestion")
    
    # Notify originator to revise
    await emit_event("approval.modified", {
        "approval_id": approval_id,
        "original_payload": request.payload,
        "modifications": modifications,
        "modifier": modifier_id
    })
    
    # The originator (e.g., Planning Service) will:
    # 1. Receive this event
    # 2. Adjust the plan/content
    # 3. Call request_approval() again with updated payload
    # 4. New approval request created (linked to original)
```
