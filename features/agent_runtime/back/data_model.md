# Agent Runtime — Data Model

## Note
Primary tables (agent_executions, agent_prompts) are defined in feature_brief.md. This file covers supplementary structures.

## Table: agent_tool_calls
```sql
CREATE TABLE agent_tool_calls (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  agent_execution_id UUID NOT NULL REFERENCES agent_executions(id),
  
  tool_name VARCHAR(50) NOT NULL,
  tool_input JSONB NOT NULL,
  tool_output JSONB,
  
  status VARCHAR(20) NOT NULL DEFAULT 'pending',
  -- Status: pending | executing | completed | failed
  
  duration_ms INTEGER,
  error TEXT,
  
  sequence_number INTEGER NOT NULL,  -- Order of tool call in execution
  
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_tool_calls_execution ON agent_tool_calls(agent_execution_id);
CREATE INDEX idx_tool_calls_tool ON agent_tool_calls(tool_name);
```

## Table: agent_output_files
```sql
CREATE TABLE agent_output_files (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  agent_execution_id UUID NOT NULL REFERENCES agent_executions(id),
  
  file_path VARCHAR(500) NOT NULL,
  operation VARCHAR(10) NOT NULL,  -- created | modified | deleted
  
  content_hash VARCHAR(64),
  size_bytes INTEGER,
  language VARCHAR(30),
  
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_output_files_execution ON agent_output_files(agent_execution_id);
```

## Table: sandbox_environments
```sql
CREATE TABLE sandbox_environments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  agent_execution_id UUID REFERENCES agent_executions(id),
  
  container_id VARCHAR(100),
  status VARCHAR(20) NOT NULL DEFAULT 'creating',
  -- Status: creating | ready | in_use | terminated | failed
  
  image VARCHAR(200) NOT NULL,
  memory_limit VARCHAR(10) NOT NULL DEFAULT '512Mi',
  cpu_limit VARCHAR(5) NOT NULL DEFAULT '1',
  
  workspace_path VARCHAR(500),
  
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  terminated_at TIMESTAMP WITH TIME ZONE
);
```

## Agent Output Schema (JSON)

Each agent must return output conforming to this schema:

```json
{
  "$schema": "agent_output_v1",
  "success": true,
  "summary": "Brief description of what was done",
  "files": [
    {
      "path": "relative/path/to/file.ts",
      "operation": "created | modified | deleted",
      "content": "full file content (for created/modified)"
    }
  ],
  "commit_message": "conventional commit message",
  "dependencies_added": ["package@version"],
  "dependencies_removed": ["package"],
  "migrations": ["migration_file_name.sql"],
  "notes": ["Any important notes for downstream steps"],
  "tokens_used": 4200,
  "tool_calls_count": 8
}
```
