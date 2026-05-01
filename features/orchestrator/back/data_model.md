# Orchestrator — Data Model

## Note
The primary data model for the Orchestrator is defined in `feature_brief.md` (tables: executions, execution_steps, execution_events). This file provides additional detail on the LangGraph state schema and Redis structures.

## LangGraph State Schema

```python
from typing import TypedDict, Literal
from langgraph.graph import StateGraph

class ExecutionState(TypedDict):
    execution_id: str
    plan_id: str
    project_id: str
    
    # DAG definition
    dag_nodes: list[dict]  # [{step_id, agent_type, depends_on, config}]
    dag_edges: list[tuple[str, str]]  # [(from_step, to_step)]
    
    # Execution tracking
    completed_steps: list[str]
    running_steps: list[str]
    failed_steps: list[str]
    skipped_steps: list[str]
    pending_steps: list[str]
    
    # Outputs from completed steps (passed to dependent steps)
    step_outputs: dict[str, dict]  # {step_id: output_data}
    
    # Budget tracking
    tokens_used: int
    cost_usd: float
    token_budget: int
    
    # Control flags
    status: Literal["running", "paused", "completed", "failed", "cancelled"]
    pause_requested: bool
    cancel_requested: bool
    
    # Error tracking
    errors: list[dict]  # [{step_id, error_type, message, retry_count}]
    
    # Timestamps
    started_at: str
    last_checkpoint_at: str
```

## Redis Structures

### Execution Lock
```
Key: exec_lock:{execution_id}
Type: String
TTL: 300 seconds (auto-release on crash)
Value: worker_id
Purpose: Ensure single worker per execution
```

### Step Queue
```
Key: exec_queue:{execution_id}
Type: List (FIFO)
Value: step_id (steps ready for dispatch)
Purpose: Queue of steps ready to execute
```

### Progress Channel (Pub/Sub)
```
Channel: exec_progress:{execution_id}
Message: JSON {step_id, status, progress_percent, eta_minutes}
Purpose: Real-time progress to WebSocket clients
```

### Rate Limiter
```
Key: rate_limit:{agent_type}:{minute_bucket}
Type: Counter
TTL: 120 seconds
Purpose: Limit API calls per agent type per minute
```

## Checkpoint Strategy

After every step completion:
1. Serialize ExecutionState to JSON
2. Store in `executions.state` column
3. Update `execution_steps` row
4. Emit checkpoint event

On recovery:
1. Load ExecutionState from database
2. Rebuild in-memory DAG
3. Identify steps that were "running" at crash time
4. Re-queue those steps (idempotent retry)
5. Resume normal execution loop
