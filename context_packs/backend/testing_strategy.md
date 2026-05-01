# Testing Strategy

## Test Pyramid
```
        /  E2E  \         (5%) - Critical user journeys
       / Integration \    (25%) - Service + DB + external
      /    Unit Tests  \  (70%) - Pure logic, fast
```

## Framework & Tools
- **Runner:** Vitest (fast, ESM-native, TypeScript)
- **HTTP testing:** Supertest
- **Mocking:** Vitest built-in mocks + MSW for HTTP
- **Database:** Testcontainers (real PostgreSQL in Docker)
- **Coverage:** Istanbul via Vitest, minimum 80%

## File Naming & Structure
```
src/
  services/
    execution.service.ts
    execution.service.test.ts      ← Unit tests (co-located)
  repositories/
    execution.repo.ts
    execution.repo.test.ts         ← Integration tests
tests/
  e2e/
    execution-flow.e2e.test.ts     ← End-to-end tests
  fixtures/
    plans.fixture.ts               ← Shared test data
  helpers/
    db.helper.ts                   ← Test DB setup/teardown
    auth.helper.ts                 ← Create test users/tokens
```

## Unit Test Pattern
```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { ExecutionService } from './execution.service';

describe('ExecutionService', () => {
  let service: ExecutionService;
  let mockRepo: MockedObject<ExecutionRepository>;
  let mockEventBus: MockedObject<EventBus>;

  beforeEach(() => {
    mockRepo = {
      create: vi.fn(),
      findById: vi.fn(),
      update: vi.fn(),
    };
    mockEventBus = { emit: vi.fn() };
    service = new ExecutionService(mockRepo, mockEventBus);
  });

  describe('startExecution', () => {
    it('should create execution with pending_approval status', async () => {
      mockRepo.create.mockResolvedValue(mockExecution);
      
      const result = await service.startExecution(validInput, actor);
      
      expect(mockRepo.create).toHaveBeenCalledWith(
        expect.objectContaining({ status: 'pending_approval' })
      );
      expect(mockEventBus.emit).toHaveBeenCalledWith(
        'execution.created',
        expect.any(Object)
      );
    });

    it('should throw if project already has active execution', async () => {
      mockRepo.findActiveByProject.mockResolvedValue([existingExecution]);
      
      await expect(service.startExecution(validInput, actor))
        .rejects.toThrow(ExecutionAlreadyRunningError);
    });
  });
});
```

## Integration Test Pattern
```typescript
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import { setupTestDB, teardownTestDB } from '../helpers/db.helper';

describe('ExecutionRepository (integration)', () => {
  let db: TestDatabase;

  beforeAll(async () => {
    db = await setupTestDB();
  });

  afterAll(async () => {
    await teardownTestDB(db);
  });

  it('should create and retrieve execution with all relations', async () => {
    const project = await createTestProject(db);
    const execution = await repo.create({
      projectId: project.id,
      planId: 'test-plan-id',
      status: 'running'
    });

    const found = await repo.findById(execution.id);
    expect(found).toMatchObject({
      id: execution.id,
      status: 'running',
      project: expect.objectContaining({ id: project.id })
    });
  });
});
```

## Coverage Requirements
| Layer | Minimum | Target |
|-------|---------|--------|
| Services | 85% | 95% |
| Repositories | 70% | 85% |
| Controllers | 60% | 80% |
| Utils/Helpers | 90% | 100% |
| Overall | 80% | 90% |

## CI Pipeline
```yaml
test:
  steps:
    - pnpm install
    - pnpm lint
    - pnpm typecheck
    - pnpm test:unit --coverage
    - pnpm test:integration
    - pnpm test:e2e (on merge to main only)
```
