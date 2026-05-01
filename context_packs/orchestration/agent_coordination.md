# Agent Coordination

## Agent Types

| Agent | Responsibility | Primary Model | Fallback |
|-------|---------------|---------------|----------|
| `planner` | Decompose goals into plans | Claude Opus | GPT-4o |
| `coder` | Write/modify source code | Claude Sonnet | DeepSeek Coder |
| `tester` | Write tests, validate output | Claude Sonnet | GPT-4o |
| `reviewer` | Code review, quality check | Claude Opus | GPT-4o |
| `docs` | Documentation generation | Claude Sonnet | GPT-4o-mini |
| `devops` | CI/CD, Docker, deployment | Claude Sonnet | GPT-4o |
| `architect` | System design decisions | Claude Opus | GPT-4o |

## Agent Selection Logic
```python
def select_agent(step: PlanStep, context: ExecutionContext) -> AgentConfig:
    """
    Select the appropriate agent based on step type, complexity, and budget.
    """
    # Base selection from step type
    agent_type = step.agent_type
    
    # Model selection based on complexity
    if step.complexity == 'high' or step.requires_reasoning:
        model = 'claude-opus'
    elif step.complexity == 'medium':
        model = 'claude-sonnet'
    else:
        model = 'claude-haiku'
    
    # Budget override: downgrade if running low
    if context.remaining_budget < context.total_budget * 0.2:
        model = downgrade_model(model)
    
    # Temperature based on task type
    temperature = {
        'coder': 0.1,      # Deterministic code
        'planner': 0.4,    # Creative planning
        'reviewer': 0.2,   # Analytical review
        'docs': 0.3,       # Natural writing
        'tester': 0.1,     # Deterministic tests
    }.get(agent_type, 0.2)
    
    return AgentConfig(
        type=agent_type,
        model=model,
        temperature=temperature,
        max_tokens=step.estimated_tokens * 1.5,
        timeout=step.timeout_seconds,
        tools=get_tools_for_agent(agent_type),
    )
```

## Context Handoff Between Agents

### Handoff Protocol
```python
class AgentHandoff:
    """
    When one agent's output becomes another's input.
    """
    
    @staticmethod
    def prepare_handoff(
        source_output: StepOutput,
        target_step: PlanStep,
        max_context_tokens: int
    ) -> HandoffPayload:
        # 1. Extract relevant portions of source output
        relevant_files = [
            f for f in source_output.files 
            if f.path in target_step.input_files
        ]
        
        # 2. Build summary if full content exceeds budget
        total_tokens = sum(count_tokens(f.content) for f in relevant_files)
        
        if total_tokens > max_context_tokens * 0.6:
            # Summarize large outputs
            summary = summarize_output(source_output, target_step.focus_areas)
            return HandoffPayload(
                mode='summary',
                summary=summary,
                key_files=relevant_files[:3],  # Top 3 most relevant
            )
        else:
            return HandoffPayload(
                mode='full',
                files=relevant_files,
                summary=source_output.summary,
            )
```

## Tool Definitions

### Coder Agent Tools
```typescript
const coderTools = [
  {
    name: 'read_file',
    description: 'Read a file from the project repository',
    parameters: { path: 'string' }
  },
  {
    name: 'write_file',
    description: 'Write content to a file (creates or overwrites)',
    parameters: { path: 'string', content: 'string' }
  },
  {
    name: 'edit_file',
    description: 'Make targeted edits to an existing file',
    parameters: { path: 'string', edits: 'array<{find, replace}>' }
  },
  {
    name: 'run_command',
    description: 'Execute a shell command (build, lint, type-check)',
    parameters: { command: 'string', timeout: 'number' }
  },
  {
    name: 'search_codebase',
    description: 'Search for patterns across the codebase',
    parameters: { query: 'string', filePattern: 'string?' }
  },
  {
    name: 'request_clarification',
    description: 'Ask for human clarification when ambiguous',
    parameters: { question: 'string', options: 'string[]?' }
  }
];
```

### Reviewer Agent Tools
```typescript
const reviewerTools = [
  {
    name: 'read_file',
    description: 'Read a file from the project repository',
    parameters: { path: 'string' }
  },
  {
    name: 'read_diff',
    description: 'Read the diff of changes made by previous agent',
    parameters: { stepId: 'string' }
  },
  {
    name: 'approve',
    description: 'Approve the changes with optional comments',
    parameters: { comments: 'string?' }
  },
  {
    name: 'request_changes',
    description: 'Request specific changes before approval',
    parameters: { issues: 'array<{file, line, severity, message}>' }
  },
  {
    name: 'escalate',
    description: 'Escalate to human reviewer for complex decisions',
    parameters: { reason: 'string', context: 'string' }
  }
];
```

## Multi-Agent Conversation

### Review Loop Pattern
```
[Coder] → produces code
    ↓
[Reviewer] → reviews, finds issues
    ↓
[Coder] → fixes issues (with review feedback as context)
    ↓
[Reviewer] → approves (or loops again, max 3 iterations)
    ↓
[Merge] → commit to repo
```

### Implementation
```python
async def review_loop(code_step: StepOutput, max_iterations: int = 3):
    """
    Iterative review until approval or max iterations reached.
    """
    current_code = code_step
    
    for iteration in range(max_iterations):
        # Review
        review = await execute_agent(
            agent_type='reviewer',
            context={
                'code_changes': current_code.files,
                'original_plan': code_step.plan_context,
                'iteration': iteration + 1,
                'previous_feedback': previous_feedback if iteration > 0 else None,
            }
        )
        
        if review.decision == 'approved':
            return MergeResult(approved=True, iterations=iteration + 1)
        
        if review.decision == 'escalate':
            return await escalate_to_human(review, current_code)
        
        # Fix issues
        previous_feedback = review.issues
        current_code = await execute_agent(
            agent_type='coder',
            context={
                'original_code': current_code.files,
                'review_feedback': review.issues,
                'instruction': 'Fix the issues identified in the review',
            }
        )
    
    # Max iterations reached — escalate
    return await escalate_to_human(review, current_code)
```

## Agent Memory (Per-Session)

```python
class AgentSession:
    """
    Maintains conversation history within a single agent execution.
    Cleared between steps. Persisted for retry/resume.
    """
    messages: list[Message] = []
    tool_calls: list[ToolCall] = []
    tokens_used: int = 0
    
    def add_message(self, role: str, content: str):
        self.messages.append(Message(role=role, content=content))
        self.tokens_used += count_tokens(content)
    
    def get_context_window(self, max_tokens: int) -> list[Message]:
        """Return messages that fit within budget, prioritizing recent."""
        # Always keep system prompt + last N messages
        # Summarize older messages if needed
        pass
```
