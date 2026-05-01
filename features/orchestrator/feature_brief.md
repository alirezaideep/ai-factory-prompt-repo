# Orchestrator — Feature Brief

## Module Identity

| Field | Value |
|-------|-------|
| Module ID | `orchestrator` |
| Layer | 5 (Execution Management) |
| Owner | Backend Team Lead |
| Status | Planned |
| Priority | Critical (execution engine) |

## Purpose

The Orchestrator is the **execution engine** of the AI Software Factory. It receives an approved Plan Schema (DAG), manages the execution of each step by dispatching work to specialized agents, handles parallelism, error recovery, state persistence, and progress reporting. Built on LangGraph for stateful, resumable workflows.

## Core Capabilities

### 1. DAG Execution Engine
- Parse Plan Schema into executable DAG
- Topological sort for execution order
- Identify parallelizable branches
- Execute steps respecting dependency order
- Track completion status per node
- Support pause/resume/cancel at any point

### 2. Agent Dispatch
- Route each step to appropriate agent (code, test, review, docs, devops)
- Provide assembled context (from Prompt Factory) to each agent
- Manage agent lifecycle (start, monitor, timeout, retry)
- Collect agent output and validate format
- Pass output between dependent steps

### 3. State Management (LangGraph)
- Persist execution state to database
- Support crash recovery (resume from last checkpoint)
- State includes: current node, completed nodes, pending nodes, agent outputs
- Snapshot state before each step (for rollback)
- Support branching (parallel execution paths)

### 4. Error Handling & Recovery
- Per-step retry with exponential backoff
- Configurable max retries per step type
- Fallback strategies:
  - Retry with different model
  - Retry with expanded context
  - Skip optional steps
  - Pause and escalate to human
- Circuit breaker for repeated failures
- Dead letter queue for unrecoverable steps

### 5. Progress Reporting
- Real-time progress via WebSocket
- Percentage completion (based on DAG topology)
- ETA calculation (based on historical step durations)
- Step-level status: pending | running | completed | failed | skipped | retrying
- Emit events for each state transition

### 6. Resource Management
- Token budget tracking per execution
- Cost accumulation in real-time
- Rate limiting for LLM API calls
- Concurrent execution limits (configurable)
- Queue management for burst workloads

## API Endpoints

### POST /api/v1/orchestrator/execute
Start execution of an approved plan.
```json
// Request
{
  "plan_id": "plan_xyz",
  "execution_config": {
    "max_parallel": 3,
    "token_budget": 100000,
    "timeout_minutes": 60,
    "retry_policy": "standard"
  }
}

// Response 202 (Accepted)
{
  "execution_id": "exec_abc",
  "status": "started",
  "total_steps": 12,
  "estimated_duration_minutes": 25
}
```

### GET /api/v1/orchestrator/executions/:id
Get execution status.
```json
{
  "execution_id": "exec_abc",
  "status": "running",
  "progress": {
    "completed": 5,
    "running": 2,
    "pending": 5,
    "failed": 0,
    "percentage": 42
  },
  "current_steps": [
    {"step_id": "step_6", "agent": "code_agent", "started_at": "..."},
    {"step_id": "step_7", "agent": "test_agent", "started_at": "..."}
  ],
  "cost_so_far_usd": 0.67,
  "tokens_used": 22400,
  "eta_minutes": 15
}
```

### POST /api/v1/orchestrator/executions/:id/pause
Pause execution (completes current steps, holds pending).

### POST /api/v1/orchestrator/executions/:id/resume
Resume paused execution.

### POST /api/v1/orchestrator/executions/:id/cancel
Cancel execution (marks remaining steps as cancelled).

### POST /api/v1/orchestrator/executions/:id/retry-step
Retry a specific failed step.
```json
{
  "step_id": "step_4",
  "strategy": "expand_context"  // retry | different_model | expand_context | manual
}
```

### GET /api/v1/orchestrator/executions/:id/logs
Get execution logs (step-by-step).

### WebSocket /ws/orchestrator/executions/:id
Real-time progress stream.

## Data Model

### Table: executions
```sql
CREATE TABLE executions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  plan_id UUID NOT NULL REFERENCES plans(id),
  project_id UUID NOT NULL REFERENCES projects(id),
  
  status VARCHAR(20) NOT NULL DEFAULT 'pending',
  -- Status: pending | running | paused | completed | failed | cancelled
  
  dag JSONB NOT NULL,
  state JSONB NOT NULL DEFAULT '{}',  -- LangGraph state snapshot
  
  config JSONB NOT NULL DEFAULT '{}',
  -- {max_parallel, token_budget, timeout_minutes, retry_policy}
  
  progress JSONB NOT NULL DEFAULT '{"completed":0,"running":0,"pending":0,"failed":0}',
  
  total_tokens_used INTEGER NOT NULL DEFAULT 0,
  total_cost_usd DECIMAL(10,4) NOT NULL DEFAULT 0,
  
  started_at TIMESTAMP WITH TIME ZONE,
  completed_at TIMESTAMP WITH TIME ZONE,
  
  started_by UUID REFERENCES users(id),
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_executions_status ON executions(status);
CREATE INDEX idx_executions_plan_id ON executions(plan_id);
```

### Table: execution_steps
```sql
CREATE TABLE execution_steps (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  execution_id UUID NOT NULL REFERENCES executions(id),
  step_id VARCHAR(50) NOT NULL,
  
  agent_type VARCHAR(30) NOT NULL,
  -- Agent: code_agent | test_agent | review_agent | docs_agent | devops_agent
  
  status VARCHAR(20) NOT NULL DEFAULT 'pending',
  -- Status: pending | queued | running | completed | failed | skipped | retrying | cancelled
  
  input_context JSONB NOT NULL DEFAULT '{}',
  output JSONB,
  error JSONB,
  
  tokens_used INTEGER DEFAULT 0,
  cost_usd DECIMAL(10,4) DEFAULT 0,
  
  retry_count INTEGER NOT NULL DEFAULT 0,
  max_retries INTEGER NOT NULL DEFAULT 3,
  
  depends_on TEXT[] DEFAULT '{}',  -- step_ids this depends on
  
  started_at TIMESTAMP WITH TIME ZONE,
  completed_at TIMESTAMP WITH TIME ZONE,
  duration_ms INTEGER,
  
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  
  UNIQUE(execution_id, step_id)
);

CREATE INDEX idx_execution_steps_execution ON execution_steps(execution_id);
CREATE INDEX idx_execution_steps_status ON execution_steps(status);
```

### Table: execution_events
```sql
CREATE TABLE execution_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  execution_id UUID NOT NULL REFERENCES executions(id),
  step_id VARCHAR(50),
  
  event_type VARCHAR(30) NOT NULL,
  -- Types: step_started | step_completed | step_failed | step_retrying |
  --        execution_paused | execution_resumed | execution_completed | 
  --        budget_warning | timeout_warning
  
  payload JSONB DEFAULT '{}',
  
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_execution_events_execution ON execution_events(execution_id);
CREATE INDEX idx_execution_events_type ON execution_events(event_type);
```

## Workflow: Execution Lifecycle

```
Plan Approved
    ↓
[Create Execution Record]
    ↓
[Parse DAG → Topological Sort]
    ↓
[Identify Ready Steps (no unmet dependencies)]
    ↓
┌─────────────────────────────────────┐
│          EXECUTION LOOP             │
│                                     │
│  [Dispatch Ready Steps to Agents]   │
│       ↓ (parallel, up to max)       │
│  [Wait for Agent Completion]        │
│       ↓                             │
│  [Validate Output]                  │
│       ↓                             │
│  [Update State + Progress]          │
│       ↓                             │
│  [Identify New Ready Steps]         │
│       ↓                             │
│  [Check Budget/Timeout/Cancel]      │
│       ↓                             │
│  [Loop until all steps done]        │
└─────────────────────────────────────┘
    ↓
[All Steps Completed?]
    ├── Yes → execution.status = 'completed'
    ├── Some Failed → execution.status = 'failed' (partial)
    └── Cancelled → execution.status = 'cancelled'
    ↓
[Emit execution.completed event]
    ↓
[Trigger Feedback Engine]
```

## Error Recovery Strategies

| Strategy | When Used | Action |
|----------|-----------|--------|
| Simple Retry | Transient errors (timeout, rate limit) | Retry same request after backoff |
| Model Upgrade | Quality issues (invalid output) | Retry with stronger model (Sonnet → Opus) |
| Context Expansion | Insufficient context | Add more files from dependency graph |
| Context Reduction | Token limit exceeded | Summarize context, keep essentials |
| Step Split | Step too complex | Break into 2 smaller steps |
| Human Escalation | Repeated failures | Pause + notify team lead |
| Skip | Optional step failed | Mark as skipped, continue |

## Configuration

```yaml
orchestrator:
  execution:
    max_parallel_steps: 5
    max_parallel_executions: 10
    default_timeout_minutes: 60
    checkpoint_interval_steps: 1  # Save state after every step
  
  retry:
    max_retries_per_step: 3
    backoff_base_seconds: 5
    backoff_multiplier: 2
    max_backoff_seconds: 60
  
  budget:
    warning_threshold_percent: 80
    hard_limit_action: "pause"  # pause | cancel
  
  agents:
    code_agent_timeout_seconds: 120
    test_agent_timeout_seconds: 90
    review_agent_timeout_seconds: 60
    docs_agent_timeout_seconds: 60
    devops_agent_timeout_seconds: 180
```

## Success Metrics

| Metric | Target |
|--------|--------|
| Execution success rate | > 90% |
| Average execution time | < 30 min for standard plans |
| Step retry rate | < 15% |
| Human escalation rate | < 5% |
| Budget overrun rate | < 2% |
| State recovery success | 100% |
