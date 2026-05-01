# Agent Runtime — Agent Execution Loop Workflow

## Overview

Each agent execution follows a ReAct (Reason + Act) loop pattern:
1. Receive task + context
2. Reason about next action
3. Execute tool call
4. Observe result
5. Repeat until task complete or limit reached

## Execution Flow

```
[Receive Dispatch from Orchestrator]
    ↓
[Create Sandbox Environment]
    ↓
[Load Agent System Prompt (from agent_prompts table)]
    ↓
[Assemble Full Prompt]
    = System Prompt
    + Context Files (from Prompt Factory)
    + Dependency Outputs (from previous steps)
    + Task Description
    + Output Schema
    ↓
[Send to LLM]
    ↓
┌─────────────────────────────────────┐
│          ReAct LOOP                 │
│                                     │
│  [LLM Response]                     │
│       ↓                             │
│  [Contains tool_call?]              │
│       ├── Yes → Execute Tool        │
│       │         → Append Result     │
│       │         → Send back to LLM  │
│       │         → Loop              │
│       └── No  → Final Answer        │
│                                     │
│  [Check Limits]                     │
│       ├── Max tool calls reached?   │
│       ├── Max tokens reached?       │
│       ├── Timeout reached?          │
│       └── Cancel requested?         │
│                                     │
└─────────────────────────────────────┘
    ↓
[Validate Output Against Schema]
    ↓
[Collect Files from Sandbox]
    ↓
[Return Structured Output to Orchestrator]
    ↓
[Terminate Sandbox]
```

## Tool Execution Safety

### File System Tools
```python
ALLOWED_PATHS = ["/workspace/src", "/workspace/tests", "/workspace/docs"]
BLOCKED_EXTENSIONS = [".env", ".key", ".pem", ".secret"]
MAX_FILE_SIZE = 100_000  # bytes

async def execute_write_file(path: str, content: str) -> str:
    # Validate path is within allowed directories
    if not any(path.startswith(p) for p in ALLOWED_PATHS):
        return "ERROR: Path not allowed"
    
    # Validate extension
    if any(path.endswith(ext) for ext in BLOCKED_EXTENSIONS):
        return "ERROR: File type not allowed"
    
    # Validate size
    if len(content.encode()) > MAX_FILE_SIZE:
        return "ERROR: File too large"
    
    # Write file
    write_to_sandbox(path, content)
    return f"OK: Written {len(content)} bytes to {path}"
```

### Command Execution Tools
```python
ALLOWED_COMMANDS = ["npm", "npx", "node", "python", "pytest", "eslint", "tsc"]
BLOCKED_COMMANDS = ["rm -rf /", "curl", "wget", "ssh", "sudo"]
MAX_EXECUTION_TIME = 30  # seconds

async def execute_command(command: str) -> str:
    # Validate command
    base_cmd = command.split()[0]
    if base_cmd not in ALLOWED_COMMANDS:
        return f"ERROR: Command '{base_cmd}' not allowed"
    
    if any(blocked in command for blocked in BLOCKED_COMMANDS):
        return "ERROR: Blocked command pattern detected"
    
    # Execute with timeout
    result = await run_in_sandbox(command, timeout=MAX_EXECUTION_TIME)
    return result.stdout[:5000]  # Truncate long outputs
```

## Context Assembly Strategy

```python
async def assemble_agent_context(
    step: PlanStep,
    plan: Plan,
    dependency_outputs: dict
) -> list[Message]:
    """
    Assemble context for agent, respecting token budget.
    Priority order:
    1. Task description (always included)
    2. Directly referenced files (from step.context_files)
    3. Dependency outputs (from previous steps)
    4. Shared conventions (backend.md or frontend.md)
    5. Related patterns (from Knowledge Base)
    """
    messages = []
    token_budget = step.config.get("max_context_tokens", 50000)
    tokens_used = 0
    
    # 1. Task (always)
    task_msg = format_task(step.task)
    messages.append(task_msg)
    tokens_used += count_tokens(task_msg)
    
    # 2. Direct context files
    for file_path in step.context_files:
        content = await repo.read_file(file_path)
        if tokens_used + count_tokens(content) > token_budget * 0.6:
            break
        messages.append({"role": "user", "content": f"FILE: {file_path}\n\n{content}"})
        tokens_used += count_tokens(content)
    
    # 3. Dependency outputs
    for dep_id, output in dependency_outputs.items():
        summary = summarize_output(output)
        if tokens_used + count_tokens(summary) > token_budget * 0.8:
            break
        messages.append({"role": "user", "content": f"OUTPUT FROM {dep_id}:\n\n{summary}"})
        tokens_used += count_tokens(summary)
    
    # 4. Shared conventions (if budget allows)
    if tokens_used < token_budget * 0.85:
        conventions = await get_relevant_conventions(step.agent_type)
        messages.append({"role": "user", "content": f"CONVENTIONS:\n\n{conventions}"})
    
    # 5. Similar patterns (if budget allows)
    if tokens_used < token_budget * 0.9:
        patterns = await knowledge_base.find_similar(step.task.description, top_k=2)
        if patterns:
            messages.append({"role": "user", "content": f"SIMILAR PATTERNS:\n\n{patterns}"})
    
    return messages
```

## Error Classification

| Error Type | Example | Recovery |
|------------|---------|----------|
| `compilation_error` | Generated code doesn't compile | Retry with error message in context |
| `test_failure` | Tests fail | Retry with test output |
| `timeout` | Agent took too long | Retry with simpler prompt |
| `rate_limit` | API rate limited | Wait and retry |
| `invalid_output` | Output doesn't match schema | Retry with stricter instructions |
| `sandbox_crash` | Container OOM/crash | Retry with smaller scope |
| `model_refusal` | Model refuses task | Try different model or rephrase |

## Metrics Collected Per Execution

```json
{
  "execution_id": "...",
  "step_id": "...",
  "agent_type": "code_agent",
  "model": "claude-3-opus",
  "input_tokens": 12000,
  "output_tokens": 3500,
  "total_tokens": 15500,
  "tool_calls": [
    {"tool": "read_file", "count": 3, "total_ms": 150},
    {"tool": "write_file", "count": 2, "total_ms": 80},
    {"tool": "run_command", "count": 1, "total_ms": 5000}
  ],
  "total_duration_ms": 45000,
  "llm_duration_ms": 38000,
  "tool_duration_ms": 5230,
  "overhead_ms": 1770,
  "files_created": 2,
  "files_modified": 1,
  "lines_of_code_generated": 180,
  "success": true
}
```
