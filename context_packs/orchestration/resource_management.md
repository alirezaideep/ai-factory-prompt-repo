# Resource Management

## Token Budget System

### Budget Hierarchy
```
Organization Budget (monthly)
  └── Project Budget (per project)
       └── Execution Budget (per execution)
            └── Step Budget (per step)
```

### Budget Allocation
```typescript
interface BudgetConfig {
  organization: {
    monthlyTokenLimit: number;      // e.g., 50_000_000
    monthlyDollarLimit: number;     // e.g., 500
    alertThresholds: number[];      // e.g., [0.5, 0.75, 0.9]
  };
  project: {
    maxTokensPerExecution: number;  // e.g., 2_000_000
    maxDollarsPerExecution: number; // e.g., 50
    maxConcurrentExecutions: number; // e.g., 3
  };
  step: {
    defaultTokenBudget: number;     // e.g., 100_000
    maxTokenBudget: number;         // e.g., 500_000
    contextWindowReserve: number;   // e.g., 0.2 (20% for output)
  };
}
```

### Cost Tracking
```python
class CostTracker:
    """Track token usage and costs in real-time."""
    
    PRICING = {
        'claude-opus': {'input': 15.0 / 1_000_000, 'output': 75.0 / 1_000_000},
        'claude-sonnet': {'input': 3.0 / 1_000_000, 'output': 15.0 / 1_000_000},
        'claude-haiku': {'input': 0.25 / 1_000_000, 'output': 1.25 / 1_000_000},
        'gpt-4o': {'input': 5.0 / 1_000_000, 'output': 15.0 / 1_000_000},
        'gpt-4o-mini': {'input': 0.15 / 1_000_000, 'output': 0.6 / 1_000_000},
        'deepseek-coder': {'input': 0.14 / 1_000_000, 'output': 0.28 / 1_000_000},
    }
    
    async def record_usage(self, execution_id: str, step_id: str, usage: TokenUsage):
        cost = (
            usage.input_tokens * self.PRICING[usage.model]['input'] +
            usage.output_tokens * self.PRICING[usage.model]['output']
        )
        
        await db.insert(token_usage).values({
            'execution_id': execution_id,
            'step_id': step_id,
            'model': usage.model,
            'input_tokens': usage.input_tokens,
            'output_tokens': usage.output_tokens,
            'cost_usd': cost,
            'timestamp': datetime.utcnow(),
        })
        
        # Check budget
        total_cost = await self.get_execution_total(execution_id)
        if total_cost > self.get_budget(execution_id):
            raise BudgetExceededError(execution_id, total_cost)
    
    async def get_execution_total(self, execution_id: str) -> float:
        result = await db.query(
            "SELECT SUM(cost_usd) as total FROM token_usage WHERE execution_id = ?",
            [execution_id]
        )
        return result.total or 0.0
```

## Concurrency Control

### Execution Slots
```python
class ExecutionScheduler:
    """Manage concurrent execution slots per project and globally."""
    
    def __init__(self, config: SchedulerConfig):
        self.max_global = config.max_global_executions  # e.g., 20
        self.max_per_project = config.max_per_project   # e.g., 3
        self.max_parallel_steps = config.max_parallel_steps  # e.g., 5
    
    async def can_start(self, project_id: str) -> bool:
        global_running = await self.count_running_global()
        project_running = await self.count_running_project(project_id)
        
        return (
            global_running < self.max_global and
            project_running < self.max_per_project
        )
    
    async def acquire_slot(self, execution_id: str, project_id: str) -> bool:
        """Atomic slot acquisition using Redis."""
        async with redis.lock(f"scheduler:{project_id}", timeout=5):
            if not await self.can_start(project_id):
                return False
            await redis.sadd(f"running:global", execution_id)
            await redis.sadd(f"running:project:{project_id}", execution_id)
            return True
    
    async def release_slot(self, execution_id: str, project_id: str):
        await redis.srem(f"running:global", execution_id)
        await redis.srem(f"running:project:{project_id}", execution_id)
```

### Queue Priority
```python
class PriorityQueue:
    """
    Execution queue with priority levels.
    Higher priority executions get slots first.
    """
    PRIORITIES = {
        'critical': 0,   # Hotfix, production issue
        'high': 1,       # Feature with deadline
        'normal': 2,     # Standard development
        'low': 3,        # Refactoring, nice-to-have
        'background': 4, # Documentation, cleanup
    }
    
    async def enqueue(self, execution_id: str, priority: str):
        score = self.PRIORITIES[priority] * 1_000_000 + time.time()
        await redis.zadd('execution_queue', {execution_id: score})
    
    async def dequeue(self) -> str | None:
        """Get highest priority (lowest score) execution."""
        results = await redis.zpopmin('execution_queue', count=1)
        return results[0] if results else None
```

## Resource Monitoring

### Metrics Collected
| Metric | Granularity | Alert Threshold |
|--------|-------------|-----------------|
| Tokens/minute | Per model | > 80% of rate limit |
| Cost/hour | Per org | > daily_budget / 8 |
| Queue depth | Global | > 50 pending |
| Execution duration | Per step | > 2x estimated |
| Error rate | Per agent type | > 20% in 5 min |
| Concurrent executions | Global | > 80% capacity |

### Auto-Scaling Rules
```yaml
scaling:
  rules:
    - metric: queue_depth
      threshold: 20
      action: increase_max_parallel
      cooldown: 300s
      
    - metric: error_rate
      threshold: 0.3
      action: decrease_max_parallel
      cooldown: 600s
      
    - metric: budget_utilization
      threshold: 0.9
      action: pause_low_priority
      cooldown: 3600s
```

## Timeout Management
```typescript
// Per-step timeout with graceful shutdown
async function executeWithTimeout(
  agent: Agent,
  context: AgentContext,
  timeoutMs: number
): Promise<StepOutput> {
  const controller = new AbortController();
  
  const timer = setTimeout(() => {
    controller.abort();
  }, timeoutMs);
  
  try {
    return await agent.execute(context, { signal: controller.signal });
  } catch (error) {
    if (error.name === 'AbortError') {
      // Save partial progress before timeout
      const partial = await agent.getPartialOutput();
      throw new TimeoutError(`Step timed out after ${timeoutMs}ms`, { partial });
    }
    throw error;
  } finally {
    clearTimeout(timer);
  }
}
```
