# Planning Service — Data Model

## Database: PostgreSQL

### Table: plans
```sql
CREATE TABLE plans (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  task_id UUID NOT NULL REFERENCES tasks(id),
  version INTEGER NOT NULL DEFAULT 1,
  status VARCHAR(20) NOT NULL DEFAULT 'generating',
  -- Status: generating | generated | approved | rejected | superseded | executing
  
  dag JSONB NOT NULL DEFAULT '{}',
  cost_estimate JSONB NOT NULL DEFAULT '{}',
  risk_assessment JSONB NOT NULL DEFAULT '{}',
  timeline JSONB NOT NULL DEFAULT '{}',
  
  affected_modules TEXT[] NOT NULL DEFAULT '{}',
  context_files_used TEXT[] NOT NULL DEFAULT '{}',
  
  feedback TEXT,  -- Human feedback that triggered this version
  parent_version INTEGER,  -- Previous version this was derived from
  
  llm_model VARCHAR(50) NOT NULL,
  llm_tokens_used INTEGER NOT NULL DEFAULT 0,
  generation_time_ms INTEGER,
  
  created_by UUID REFERENCES users(id),
  approved_by UUID REFERENCES users(id),
  approved_at TIMESTAMP WITH TIME ZONE,
  
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  
  UNIQUE(task_id, version)
);

CREATE INDEX idx_plans_task_id ON plans(task_id);
CREATE INDEX idx_plans_status ON plans(status);
CREATE INDEX idx_plans_created_at ON plans(created_at DESC);
```

### Table: tasks
```sql
CREATE TABLE tasks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title VARCHAR(255) NOT NULL,
  description TEXT,
  goal TEXT NOT NULL,
  
  scope JSONB NOT NULL DEFAULT '{}',
  -- {"modules": [...], "constraints": [...]}
  
  priority VARCHAR(10) NOT NULL DEFAULT 'medium',
  -- Priority: critical | high | medium | low
  
  status VARCHAR(20) NOT NULL DEFAULT 'draft',
  -- Status: draft | submitted | planning | plan_ready | approving | approved | executing | completed | failed | cancelled
  
  attachments JSONB DEFAULT '[]',
  metadata JSONB DEFAULT '{}',
  
  submitted_by UUID NOT NULL REFERENCES users(id),
  project_id UUID REFERENCES projects(id),
  
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_tasks_status ON tasks(status);
CREATE INDEX idx_tasks_priority ON tasks(priority);
CREATE INDEX idx_tasks_submitted_by ON tasks(submitted_by);
CREATE INDEX idx_tasks_project_id ON tasks(project_id);
```

### Table: plan_comments
```sql
CREATE TABLE plan_comments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  plan_id UUID NOT NULL REFERENCES plans(id),
  step_id VARCHAR(50),  -- NULL means comment on whole plan
  author_id UUID NOT NULL REFERENCES users(id),
  content TEXT NOT NULL,
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_plan_comments_plan_id ON plan_comments(plan_id);
```

### Table: plan_history
```sql
CREATE TABLE plan_history (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  plan_id UUID NOT NULL REFERENCES plans(id),
  action VARCHAR(20) NOT NULL,
  -- Action: generated | approved | rejected | modified | regenerated
  actor_id UUID REFERENCES users(id),
  details JSONB DEFAULT '{}',
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_plan_history_plan_id ON plan_history(plan_id);
```

## Redis Cache

### Plan Generation Lock
```
Key: plan_lock:{task_id}
TTL: 120 seconds
Value: {generating_plan_id}
Purpose: Prevent duplicate generation for same task
```

### Cost Estimation Cache
```
Key: cost_cache:{model}:{token_count_bucket}
TTL: 1 hour
Value: {estimated_usd}
Purpose: Cache pricing calculations
```

### Historical Plans Index
```
Key: plan_index:{module_name}
Type: Sorted Set
Score: timestamp
Value: plan_id
Purpose: Quick lookup of plans affecting a module
```
