# Service Layer Patterns

## Architecture Overview
```
Controller (HTTP/WS) → Service (Business Logic) → Repository (Data Access)
                     ↗ Event Bus (Side Effects)
```

## Service Structure
```typescript
// services/execution.service.ts
export class ExecutionService {
  constructor(
    private readonly executionRepo: ExecutionRepository,
    private readonly stepRepo: StepRepository,
    private readonly eventBus: EventBus,
    private readonly orchestrator: OrchestratorClient,
    private readonly logger: Logger
  ) {}

  async startExecution(input: StartExecutionInput, actor: Actor): Promise<Execution> {
    // 1. Validate business rules
    await this.validateCanStart(input, actor);
    
    // 2. Perform operation
    const execution = await this.executionRepo.create({
      ...input,
      status: 'pending_approval',
      created_by: actor.id
    });
    
    // 3. Emit event (side effects handled by listeners)
    await this.eventBus.emit('execution.created', {
      executionId: execution.id,
      projectId: input.projectId,
      actor
    });
    
    // 4. Return result
    return execution;
  }
}
```

## Dependency Injection
```typescript
// Use tsyringe or similar DI container
import { container, injectable, inject } from 'tsyringe';

@injectable()
export class PlanningService {
  constructor(
    @inject('PlanRepository') private planRepo: PlanRepository,
    @inject('LLMClient') private llm: LLMClient,
    @inject('EventBus') private events: EventBus
  ) {}
}
```

## Transaction Management
```typescript
// For operations spanning multiple repositories
async function transferOwnership(projectId: string, newOwnerId: string, actor: Actor) {
  return await db.transaction(async (tx) => {
    // All operations use the same transaction
    const project = await projectRepo.findById(projectId, { tx });
    
    if (!project) throw new NotFoundError('Project');
    if (project.owner_id !== actor.id) throw new ForbiddenError();
    
    await projectRepo.update(projectId, { owner_id: newOwnerId }, { tx });
    await memberRepo.updateRole(newOwnerId, projectId, 'owner', { tx });
    await auditRepo.log('ownership_transfer', { projectId, from: actor.id, to: newOwnerId }, { tx });
    
    return project;
  });
}
```

## Event-Driven Side Effects
```typescript
// Decouple side effects from main logic
// listeners/execution.listeners.ts

eventBus.on('execution.completed', async (event) => {
  // These run asynchronously, don't block the main flow
  await feedbackEngine.extractSignals(event.executionId);
  await notificationService.notify(event.actor, 'Execution completed');
  await metricsService.recordCompletion(event);
});

eventBus.on('execution.failed', async (event) => {
  await alertService.createAlert('execution_failure', event);
  await notificationService.notify(event.actor, 'Execution failed', 'error');
});
```

## Error Handling in Services
```typescript
// Use domain-specific errors
export class DomainError extends Error {
  constructor(
    message: string,
    public readonly code: string,
    public readonly statusCode: number = 400,
    public readonly details?: Record<string, any>
  ) {
    super(message);
  }
}

export class ExecutionAlreadyRunningError extends DomainError {
  constructor(projectId: string) {
    super(
      `Project ${projectId} already has an active execution`,
      'EXECUTION_ALREADY_RUNNING',
      409,
      { projectId }
    );
  }
}
```

## Validation Layer
```typescript
// Use Zod for input validation at service boundary
import { z } from 'zod';

const StartExecutionSchema = z.object({
  planId: z.string().uuid(),
  projectId: z.string().uuid(),
  config: z.object({
    maxParallel: z.number().int().min(1).max(10).default(3),
    retryPolicy: z.object({
      maxRetries: z.number().int().min(0).max(5).default(2),
      backoffSeconds: z.array(z.number()).default([30, 120])
    }).optional(),
    timeoutMinutes: z.number().int().min(1).max(480).default(120),
    budgetLimitUsd: z.number().positive().max(1000).optional()
  })
});

type StartExecutionInput = z.infer<typeof StartExecutionSchema>;
```

## Caching Strategy
```typescript
// Cache at service level, invalidate on writes
class ProjectService {
  private cache = new CacheManager('projects', { ttl: 300 });

  async getById(id: string): Promise<Project> {
    return this.cache.getOrSet(id, () => this.repo.findById(id));
  }

  async update(id: string, data: Partial<Project>): Promise<Project> {
    const result = await this.repo.update(id, data);
    await this.cache.invalidate(id);
    return result;
  }
}
```
