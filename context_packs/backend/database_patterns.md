# Database Patterns

## Technology Stack
- **Primary DB:** PostgreSQL 16 (relational data, JSONB for flexible schemas)
- **Vector DB:** Qdrant (embeddings, semantic search)
- **Cache:** Redis (sessions, rate limiting, pub/sub)
- **ORM:** Drizzle ORM (TypeScript, type-safe queries)

## Schema Design Principles

### Naming
- Tables: plural, snake_case (`execution_steps`, `approval_requests`)
- Columns: snake_case (`created_at`, `project_id`)
- Primary keys: `id` (UUID v7 for time-ordering)
- Foreign keys: `{referenced_table_singular}_id` (`project_id`, `user_id`)
- Indexes: `idx_{table}_{columns}` (`idx_orders_status_created`)
- Constraints: `{type}_{table}_{columns}` (`uq_users_email`, `fk_orders_project`)

### Standard Columns (every table)
```sql
id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
created_by UUID REFERENCES users(id),
version INTEGER NOT NULL DEFAULT 1  -- Optimistic locking
```

### Soft Delete Pattern
```sql
deleted_at TIMESTAMPTZ,  -- NULL = active, timestamp = deleted
deleted_by UUID REFERENCES users(id)
```

### Audit Trail Pattern
```sql
CREATE TABLE audit_log (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  table_name TEXT NOT NULL,
  record_id UUID NOT NULL,
  action TEXT NOT NULL,  -- INSERT, UPDATE, DELETE
  old_values JSONB,
  new_values JSONB,
  changed_fields TEXT[],
  performed_by UUID REFERENCES users(id),
  performed_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  ip_address INET,
  user_agent TEXT
);
```

## Migration Strategy

### File Naming
```
migrations/
  0001_create_users.sql
  0002_create_projects.sql
  0003_add_status_to_projects.sql
  0004_create_executions.sql
```

### Migration Rules
1. Each migration is a single SQL file (up only, no down)
2. Migrations are immutable once applied
3. Always use `IF NOT EXISTS` for CREATE
4. Always use `IF EXISTS` for DROP
5. Large data migrations: separate from schema changes
6. Add indexes CONCURRENTLY to avoid locks

### Migration Template
```sql
-- Migration: 0005_add_priority_to_tasks
-- Description: Add priority column with default value
-- Breaking: No
-- Requires backfill: No

BEGIN;

ALTER TABLE tasks 
  ADD COLUMN IF NOT EXISTS priority TEXT NOT NULL DEFAULT 'medium'
  CHECK (priority IN ('low', 'medium', 'high', 'critical'));

CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_tasks_priority 
  ON tasks(priority) WHERE deleted_at IS NULL;

COMMIT;
```

## Query Patterns

### Repository Pattern
```typescript
// Each table has a repository with standard CRUD + custom queries
class ExecutionRepository {
  async findById(id: string): Promise<Execution | null>
  async findMany(filters: ExecutionFilters, pagination: Pagination): Promise<PaginatedResult<Execution>>
  async create(data: CreateExecution): Promise<Execution>
  async update(id: string, data: Partial<Execution>): Promise<Execution>
  async softDelete(id: string, deletedBy: string): Promise<void>
  
  // Custom queries
  async findActiveByProject(projectId: string): Promise<Execution[]>
  async getStatsByDateRange(start: Date, end: Date): Promise<ExecutionStats>
}
```

### Optimistic Locking
```typescript
async function updateWithLock(id: string, data: Partial<Entity>, expectedVersion: number) {
  const result = await db.update(table)
    .set({ ...data, version: expectedVersion + 1, updated_at: new Date() })
    .where(and(eq(table.id, id), eq(table.version, expectedVersion)))
    .returning();
  
  if (result.length === 0) {
    throw new ConflictError('Record was modified by another process');
  }
  return result[0];
}
```

### JSONB Usage
```sql
-- For flexible/evolving schemas (agent configs, plan payloads)
config JSONB NOT NULL DEFAULT '{}',

-- Query JSONB
SELECT * FROM executions 
WHERE config->>'model' = 'claude-3-opus'
AND (config->'retry_policy'->>'max_retries')::int > 2;

-- Index JSONB paths
CREATE INDEX idx_executions_config_model 
ON executions ((config->>'model'));
```

## Performance Guidelines

### Indexing Strategy
- Always index foreign keys
- Composite indexes: most selective column first
- Partial indexes for filtered queries: `WHERE deleted_at IS NULL`
- GIN indexes for JSONB and array columns
- Monitor with `pg_stat_user_indexes`

### Connection Pooling
- Use PgBouncer in transaction mode
- Pool size: 2x CPU cores for write-heavy, 4x for read-heavy
- Statement timeout: 30s default, 120s for reports

### Query Optimization Rules
1. Never SELECT * in production code
2. Limit result sets (max 100 per query)
3. Use cursor-based pagination for large datasets
4. Batch inserts (max 1000 per batch)
5. Use EXPLAIN ANALYZE for queries > 100ms
