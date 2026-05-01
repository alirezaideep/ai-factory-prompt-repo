# Agent Runtime — Feature Brief

## Module Identity

| Field | Value |
|-------|-------|
| Module ID | `agent_runtime` |
| Layer | 6 (Execution) |
| Owner | Backend Team Lead |
| Status | Planned |
| Priority | Critical (code generation) |

## Purpose

The Agent Runtime provides the execution environment for specialized AI agents. Each agent type has a specific role (code generation, testing, review, documentation, DevOps) and operates within a sandboxed environment with access to tools, context, and the repository. Agents consume assembled prompts and produce structured outputs.

## Agent Types

### 1. Code Agent
- **Role:** Generate, modify, or refactor code
- **Model:** Claude Opus (primary), Claude Sonnet (fallback)
- **Tools:** File system (read/write), terminal (execute), git (commit)
- **Input:** Context files + task description + existing code (if modification)
- **Output:** Code files + commit message + dependency changes
- **Constraints:** Must follow conventions from shared/backend.md or shared/frontend.md

### 2. Test Agent
- **Role:** Generate unit tests, integration tests, E2E tests
- **Model:** Claude Sonnet (primary)
- **Tools:** File system, terminal (run tests), coverage tools
- **Input:** Code files (from Code Agent) + test conventions + data model
- **Output:** Test files + test results + coverage report
- **Constraints:** Minimum 80% coverage, must test edge cases

### 3. Review Agent
- **Role:** Code review, security audit, best practice validation
- **Model:** Claude Opus (primary)
- **Tools:** File system (read-only), static analysis tools
- **Input:** Code diff + conventions + security checklist
- **Output:** Review comments + severity ratings + approval/rejection
- **Constraints:** Must check security, performance, maintainability

### 4. Docs Agent
- **Role:** Generate/update documentation, API docs, changelogs
- **Model:** Claude Sonnet (primary)
- **Tools:** File system, markdown renderer
- **Input:** Code changes + existing docs + changelog format
- **Output:** Updated docs + changelog entry + API reference updates
- **Constraints:** Must maintain consistency with existing docs style

### 5. DevOps Agent
- **Role:** Infrastructure changes, CI/CD, deployment configs
- **Model:** Claude Sonnet (primary)
- **Tools:** File system, terminal, Docker, K8s CLI
- **Input:** Infrastructure requirements + existing configs
- **Output:** Config files + deployment scripts + migration scripts
- **Constraints:** Must validate configs before output, no destructive operations

## Architecture

### Agent Execution Flow
```
[Orchestrator] → dispatch(step, context)
    ↓
[Agent Runtime]
    ↓
[Select Agent Type based on step.agent]
    ↓
[Load Agent System Prompt]
    ↓
[Assemble Full Prompt = System + Context + Task]
    ↓
[Create Sandbox Environment]
    ↓
[Execute Agent (LLM + Tools loop)]
    ↓
[Validate Output Schema]
    ↓
[Return Structured Output to Orchestrator]
```

### Sandbox Environment
Each agent execution runs in an isolated environment:
```
/workspace/
├── src/           ← Project source code (mounted read-write for code agent, read-only for others)
├── tests/         ← Test files
├── docs/          ← Documentation
├── .context/      ← Assembled context files (read-only)
├── .output/       ← Agent writes output here
└── .tools/        ← Available CLI tools
```

### Tool Definitions
```python
AGENT_TOOLS = {
    "code_agent": [
        {"name": "read_file", "description": "Read a file from workspace"},
        {"name": "write_file", "description": "Write/create a file in workspace"},
        {"name": "run_command", "description": "Execute shell command"},
        {"name": "search_code", "description": "Search codebase with regex"},
        {"name": "git_diff", "description": "Show current changes"},
        {"name": "git_commit", "description": "Commit current changes"},
    ],
    "test_agent": [
        {"name": "read_file", "description": "Read a file from workspace"},
        {"name": "write_file", "description": "Write test files"},
        {"name": "run_tests", "description": "Execute test suite"},
        {"name": "coverage_report", "description": "Generate coverage report"},
    ],
    "review_agent": [
        {"name": "read_file", "description": "Read a file from workspace"},
        {"name": "git_diff", "description": "Show changes to review"},
        {"name": "run_linter", "description": "Run static analysis"},
        {"name": "add_comment", "description": "Add review comment"},
    ],
    "docs_agent": [
        {"name": "read_file", "description": "Read a file"},
        {"name": "write_file", "description": "Write documentation file"},
        {"name": "list_files", "description": "List directory contents"},
    ],
    "devops_agent": [
        {"name": "read_file", "description": "Read config files"},
        {"name": "write_file", "description": "Write config files"},
        {"name": "run_command", "description": "Execute infrastructure commands"},
        {"name": "validate_config", "description": "Validate YAML/JSON configs"},
    ]
}
```

## API Endpoints

### POST /api/v1/agents/execute
Execute an agent for a specific step.
```json
// Request
{
  "execution_id": "exec_abc",
  "step_id": "step_3",
  "agent_type": "code_agent",
  "task": {
    "description": "Add priority field to orders table and update API",
    "requirements": ["Add VARCHAR(10) priority column", "Update POST /orders endpoint"],
    "acceptance_criteria": ["Migration runs without error", "API accepts priority param"]
  },
  "context_files": [
    {"path": "features/orders/back/data_model.md", "content": "..."},
    {"path": "features/orders/back/api_endpoints.md", "content": "..."},
    {"path": "shared/backend.md", "content": "..."}
  ],
  "config": {
    "model": "claude-3-opus",
    "max_tokens": 8000,
    "max_tool_calls": 20,
    "timeout_seconds": 120
  }
}

// Response 200
{
  "success": true,
  "output": {
    "files_created": ["src/migrations/003_add_priority.sql"],
    "files_modified": ["src/routes/orders.ts", "src/models/order.ts"],
    "commit_message": "feat(orders): add priority field with migration",
    "summary": "Added priority column (VARCHAR 10, default 'normal') with migration and updated API endpoint",
    "tokens_used": 4200,
    "tool_calls": 8,
    "duration_ms": 45000
  }
}
```

### GET /api/v1/agents/status/:executionId/:stepId
Get agent execution status (for long-running agents).

### POST /api/v1/agents/cancel/:executionId/:stepId
Cancel a running agent.

## Data Model

### Table: agent_executions
```sql
CREATE TABLE agent_executions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  execution_id UUID NOT NULL REFERENCES executions(id),
  step_id VARCHAR(50) NOT NULL,
  
  agent_type VARCHAR(30) NOT NULL,
  model VARCHAR(50) NOT NULL,
  
  status VARCHAR(20) NOT NULL DEFAULT 'running',
  -- Status: running | completed | failed | cancelled | timeout
  
  input_tokens INTEGER NOT NULL DEFAULT 0,
  output_tokens INTEGER NOT NULL DEFAULT 0,
  total_tokens INTEGER NOT NULL DEFAULT 0,
  cost_usd DECIMAL(10,4) DEFAULT 0,
  
  tool_calls_count INTEGER NOT NULL DEFAULT 0,
  tool_calls_log JSONB DEFAULT '[]',
  
  output JSONB,
  error JSONB,
  
  started_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  completed_at TIMESTAMP WITH TIME ZONE,
  duration_ms INTEGER,
  
  UNIQUE(execution_id, step_id)
);

CREATE INDEX idx_agent_exec_execution ON agent_executions(execution_id);
CREATE INDEX idx_agent_exec_type ON agent_executions(agent_type);
CREATE INDEX idx_agent_exec_status ON agent_executions(status);
```

### Table: agent_prompts
```sql
CREATE TABLE agent_prompts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  agent_type VARCHAR(30) NOT NULL,
  version INTEGER NOT NULL DEFAULT 1,
  
  system_prompt TEXT NOT NULL,
  tool_definitions JSONB NOT NULL,
  output_schema JSONB NOT NULL,
  
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  
  UNIQUE(agent_type, version)
);
```

## Agent System Prompts (Templates)

### Code Agent System Prompt
```
You are a senior software engineer working in a production codebase.
You follow the project conventions strictly.

CONVENTIONS:
{context_from_shared_backend_or_frontend}

CURRENT DATA MODEL:
{context_from_feature_data_model}

EXISTING API:
{context_from_feature_api_endpoints}

YOUR TASK:
{task_description}

REQUIREMENTS:
{requirements_list}

RULES:
1. Follow existing code style exactly
2. Write production-quality code (error handling, validation, types)
3. Include inline comments for complex logic
4. Create database migrations for schema changes
5. Update API documentation inline
6. Do NOT break existing functionality
7. Commit with conventional commit message format
```

### Review Agent System Prompt
```
You are a senior code reviewer. Review the following changes for:
1. Security vulnerabilities (SQL injection, XSS, auth bypass)
2. Performance issues (N+1 queries, memory leaks, blocking operations)
3. Code quality (naming, structure, DRY, SOLID)
4. Test coverage (are edge cases tested?)
5. Convention compliance (matches project standards?)

SEVERITY LEVELS:
- critical: Must fix before merge (security, data loss)
- major: Should fix (performance, maintainability)
- minor: Nice to fix (style, naming)
- suggestion: Optional improvement

OUTPUT FORMAT:
For each issue, provide:
- File and line number
- Severity
- Description
- Suggested fix
```

## Configuration

```yaml
agent_runtime:
  models:
    code_agent: "claude-3-opus"
    test_agent: "claude-3-sonnet"
    review_agent: "claude-3-opus"
    docs_agent: "claude-3-sonnet"
    devops_agent: "claude-3-sonnet"
  
  limits:
    max_tool_calls_per_execution: 30
    max_tokens_per_execution: 16000
    execution_timeout_seconds: 180
    max_file_size_bytes: 100000
  
  sandbox:
    base_image: "factory-agent-sandbox:latest"
    memory_limit: "512Mi"
    cpu_limit: "1"
    network_policy: "restricted"  # Only access to LLM API
  
  fallback:
    enabled: true
    primary_to_fallback:
      "claude-3-opus": "claude-3-sonnet"
      "claude-3-sonnet": "claude-3-haiku"
```

## Success Metrics

| Metric | Target |
|--------|--------|
| Agent success rate (first attempt) | > 85% |
| Code agent output compiles | > 95% |
| Test agent coverage achieved | > 80% |
| Review agent false positive rate | < 10% |
| Average execution time | < 60 seconds |
| Token efficiency (output/input ratio) | > 0.3 |
