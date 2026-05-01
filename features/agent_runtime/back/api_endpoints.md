# Agent Runtime — API Endpoints

## Base Path: `/api/v1/agents`

### Agent Registry

#### GET /api/v1/agents
List all registered agent types with their capabilities and status.

**Response:**
```json
{
  "data": [
    {
      "id": "code-gen-v2",
      "name": "Code Generator",
      "type": "code_generation",
      "model": "claude-sonnet-4",
      "status": "active",
      "capabilities": ["typescript", "python", "sql", "testing"],
      "maxConcurrent": 5,
      "avgTokensPerTask": 45000,
      "successRate": 0.94
    }
  ],
  "meta": { "total": 8 }
}
```

#### POST /api/v1/agents
Register a new agent type.

**Request:**
```json
{
  "name": "Code Generator v3",
  "type": "code_generation",
  "model": "claude-opus-4",
  "systemPrompt": "You are an expert software engineer...",
  "tools": ["file_write", "file_read", "shell_exec", "test_run"],
  "maxConcurrent": 3,
  "tokenBudget": 100000,
  "retryPolicy": { "maxRetries": 3, "backoff": "exponential" }
}
```

#### GET /api/v1/agents/{agentId}/metrics
Get performance metrics for a specific agent type.

**Response:**
```json
{
  "agentId": "code-gen-v2",
  "period": "last_30_days",
  "metrics": {
    "totalExecutions": 342,
    "successRate": 0.94,
    "avgDuration": 45.2,
    "avgTokensUsed": 42000,
    "avgCostPerTask": 0.63,
    "selfFixRate": 0.72,
    "escalationRate": 0.06
  }
}
```

### Agent Execution

#### POST /api/v1/agents/execute
Execute a task using a specific agent. Called by the Orchestrator.

**Request:**
```json
{
  "executionId": "exec-uuid",
  "stepId": "step-uuid",
  "agentType": "code_generation",
  "context": {
    "files": ["shared/backend.md", "features/orchestrator/back/api_endpoints.md"],
    "instruction": "Implement the /pause endpoint for executions",
    "constraints": { "maxTokens": 80000, "timeout": 300 }
  },
  "tools": ["file_write", "file_read", "shell_exec"],
  "callbackUrl": "/api/v1/orchestrator/steps/{stepId}/complete"
}
```

**Response:**
```json
{
  "taskId": "task-uuid",
  "status": "running",
  "estimatedDuration": 60
}
```

#### GET /api/v1/agents/tasks/{taskId}
Check status of a running agent task.

#### POST /api/v1/agents/tasks/{taskId}/cancel
Cancel a running agent task.

### Agent Configuration

#### PUT /api/v1/agents/{agentId}/config
Update agent configuration (system prompt, tools, budget).

#### GET /api/v1/agents/{agentId}/history
Get execution history for an agent type with filtering.

**Query Params:** `?status=failed&from=2025-01-01&limit=50`
