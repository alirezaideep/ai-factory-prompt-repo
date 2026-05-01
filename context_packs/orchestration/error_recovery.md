# Error Recovery

## Recovery Strategies

| Error Type | Strategy | Max Retries | Backoff |
|-----------|----------|-------------|---------|
| LLM timeout | Retry with same context | 3 | 30s, 60s, 120s |
| LLM rate limit | Retry with backoff | 5 | Exponential |
| LLM content filter | Rephrase prompt | 2 | Immediate |
| Invalid output format | Retry with stricter prompt | 2 | Immediate |
| Syntax error in code | Self-fix loop | 3 | Immediate |
| Test failure | Fix-and-retest loop | 3 | Immediate |
| Merge conflict | Resolve and retry | 2 | Immediate |
| Budget exceeded | Pause + notify human | 0 | N/A |
| Agent crash | Restart from checkpoint | 2 | 30s |
| Network error | Retry | 5 | Exponential |

## Retry Policy Configuration
```typescript
interface RetryPolicy {
  maxRetries: number;
  backoffType: 'fixed' | 'exponential' | 'linear';
  initialDelay: number;  // seconds
  maxDelay: number;      // seconds
  retryableErrors: string[];
  onRetry?: (attempt: number, error: Error) => void;
  onExhausted?: (error: Error) => 'skip' | 'fail' | 'escalate';
}

const DEFAULT_POLICIES: Record<string, RetryPolicy> = {
  llm_call: {
    maxRetries: 3,
    backoffType: 'exponential',
    initialDelay: 30,
    maxDelay: 300,
    retryableErrors: ['TIMEOUT', 'RATE_LIMIT', 'SERVICE_UNAVAILABLE'],
    onExhausted: () => 'escalate',
  },
  code_validation: {
    maxRetries: 3,
    backoffType: 'fixed',
    initialDelay: 0,
    maxDelay: 0,
    retryableErrors: ['SYNTAX_ERROR', 'TYPE_ERROR', 'TEST_FAILURE'],
    onExhausted: () => 'escalate',
  },
  file_operation: {
    maxRetries: 5,
    backoffType: 'exponential',
    initialDelay: 1,
    maxDelay: 30,
    retryableErrors: ['IO_ERROR', 'LOCK_CONFLICT'],
    onExhausted: () => 'fail',
  },
};
```

## Self-Fix Loop
```python
async def self_fix_loop(
    agent: Agent,
    original_output: StepOutput,
    error: ValidationError,
    max_attempts: int = 3
) -> StepOutput:
    """
    Agent attempts to fix its own output based on error feedback.
    """
    current_output = original_output
    current_error = error
    
    for attempt in range(max_attempts):
        # Build fix prompt
        fix_context = {
            'original_code': current_output.files,
            'error_message': str(current_error),
            'error_type': current_error.type,
            'attempt': attempt + 1,
            'instruction': f'Fix the following error in your output: {current_error.message}',
        }
        
        # Add error-specific hints
        if current_error.type == 'SYNTAX_ERROR':
            fix_context['hint'] = f'Line {current_error.line}: {current_error.details}'
        elif current_error.type == 'TEST_FAILURE':
            fix_context['test_output'] = current_error.test_output
            fix_context['hint'] = 'Review the failing test and fix the implementation'
        
        # Execute fix
        fixed_output = await agent.execute(fix_context)
        
        # Validate
        validation = await validate_output(fixed_output)
        if validation.valid:
            return fixed_output
        
        current_output = fixed_output
        current_error = validation.errors[0]
    
    # Exhausted — escalate
    raise SelfFixExhaustedError(
        original_error=error,
        last_error=current_error,
        attempts=max_attempts
    )
```

## Human Escalation
```python
async def escalate_to_human(
    execution_id: str,
    step_id: str,
    reason: str,
    context: dict
) -> EscalationResult:
    """
    Pause execution and request human intervention.
    """
    # 1. Pause the step
    await transition(step_id, 'step', 'running', 'paused', 'escalation')
    
    # 2. Create approval request
    approval = await create_approval_request(
        type='escalation',
        execution_id=execution_id,
        step_id=step_id,
        reason=reason,
        context=context,
        options=['fix_and_continue', 'skip_step', 'abort_execution', 'provide_guidance'],
    )
    
    # 3. Notify relevant humans
    await notify_team_leads(approval)
    
    # 4. Wait for response (with timeout)
    response = await wait_for_approval(approval.id, timeout_hours=24)
    
    if response.decision == 'fix_and_continue':
        # Human provides fix, resume
        return EscalationResult(action='resume', context=response.data)
    elif response.decision == 'skip_step':
        await transition(step_id, 'step', 'paused', 'skipped', 'human_decision')
        return EscalationResult(action='skip')
    elif response.decision == 'abort_execution':
        await transition(execution_id, 'execution', 'running', 'failed', 'human_abort')
        return EscalationResult(action='abort')
    elif response.decision == 'provide_guidance':
        # Resume with additional context from human
        return EscalationResult(action='retry_with_guidance', context=response.data)
```

## Fallback Model Strategy
```python
MODEL_FALLBACK_CHAIN = {
    'claude-opus': ['gpt-4o', 'claude-sonnet'],
    'claude-sonnet': ['gpt-4o', 'deepseek-coder'],
    'gpt-4o': ['claude-sonnet', 'gpt-4o-mini'],
}

async def execute_with_fallback(prompt: str, primary_model: str, **kwargs):
    """Try primary model, fall back to alternatives on failure."""
    models_to_try = [primary_model] + MODEL_FALLBACK_CHAIN.get(primary_model, [])
    
    for model in models_to_try:
        try:
            return await call_model(model, prompt, **kwargs)
        except (TimeoutError, RateLimitError, ServiceUnavailableError) as e:
            logger.warn(f"Model {model} failed: {e}, trying next")
            continue
    
    raise AllModelsFailedError(f"All models failed for prompt")
```

## Dead Letter Queue
```python
# Steps that fail after all retries go to DLQ for manual review
async def send_to_dlq(step: FailedStep):
    await db.insert(dead_letter_queue).values({
        'step_id': step.id,
        'execution_id': step.execution_id,
        'error': step.last_error,
        'attempts': step.retry_count,
        'context_snapshot': step.context,
        'created_at': datetime.utcnow(),
        'status': 'pending_review',
    })
    
    await notify_admins(f"Step {step.name} moved to DLQ after {step.retry_count} failures")
```
