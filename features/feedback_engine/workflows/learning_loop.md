# Feedback Engine — Learning Loop Workflow

## Overview

The Feedback Engine closes the loop between execution outcomes and future planning. It continuously learns from successes and failures to improve estimates, prompt quality, and agent selection.

## Learning Loop Flow

```
[Execution Outcome Recorded]
    ↓
[Classify Outcome]
    ├── Full success (no human correction needed)
    ├── Partial success (minor corrections)
    ├── Failure recovered (retry succeeded)
    └── Total failure (human takeover)
    ↓
[Extract Signals]
    ├── Cost accuracy: estimated vs actual
    ├── Time accuracy: estimated vs actual
    ├── Quality score: based on corrections needed
    ├── Agent performance: success rate by type
    └── Prompt effectiveness: which context helped
    ↓
[Update Models]
    ├── Calibration factors (cost/time estimates)
    ├── Agent selection weights
    ├── Context relevance scores
    └── Risk assessment accuracy
    ↓
[Generate Insights]
    ├── Improvement alerts (quality drops)
    ├── Cost optimization suggestions
    ├── Prompt refinement recommendations
    └── Process bottleneck identification
    ↓
[Surface to Command Room]
    ↓
[Human reviews and acts on insights]
```

## Signal Extraction

### From Successful Executions
```python
async def extract_success_signals(execution_id: str):
    outcome = await db.get_execution_outcome(execution_id)
    steps = await db.get_execution_steps(execution_id)
    
    signals = {
        "cost_ratio": outcome.actual_cost / outcome.estimated_cost,
        "time_ratio": outcome.actual_duration / outcome.estimated_duration,
        "token_efficiency": outcome.useful_tokens / outcome.total_tokens,
        "first_attempt_rate": sum(1 for s in steps if s.attempts == 1) / len(steps),
        "parallel_utilization": calculate_parallelism(steps),
        "context_tokens_per_step": mean([s.context_tokens for s in steps]),
    }
    
    # Store for calibration
    await db.store_signals(execution_id, signals)
    
    # Update calibration model
    await calibrator.update(outcome)
    
    return signals
```

### From Failures
```python
async def extract_failure_signals(execution_id: str, failed_step_id: str):
    step = await db.get_step(failed_step_id)
    
    failure_analysis = {
        "failure_type": classify_failure(step.error),
        "agent_type": step.agent_type,
        "model": step.model,
        "context_tokens": step.context_tokens,
        "attempt_count": step.attempts,
        "was_recovered": step.final_status == "completed",
        "recovery_method": step.recovery_method,  # retry | human | skip
        "root_cause": await analyze_root_cause(step),
    }
    
    # Pattern: if same failure type repeats > 3 times, create alert
    similar_failures = await db.count_similar_failures(
        failure_analysis["failure_type"],
        failure_analysis["agent_type"],
        days=7
    )
    
    if similar_failures >= 3:
        await create_improvement_alert(
            type="recurring_failure",
            severity="warning",
            details=failure_analysis
        )
    
    return failure_analysis
```

### From Human Corrections
```python
async def extract_correction_signals(execution_id: str, correction: HumanCorrection):
    """
    When a human corrects AI output, extract what went wrong.
    """
    signals = {
        "correction_type": correction.type,
        # Types: logic_error | style_mismatch | missing_requirement | 
        #        wrong_pattern | security_issue | performance_issue
        
        "severity": correction.severity,  # minor | moderate | major
        "original_output": correction.original,
        "corrected_output": correction.corrected,
        "affected_files": correction.files,
        "correction_explanation": correction.explanation,
    }
    
    # Store as negative example in Knowledge Base
    await knowledge_base.store_correction(
        original=correction.original,
        corrected=correction.corrected,
        explanation=correction.explanation,
        tags=["correction", correction.type, correction.agent_type]
    )
    
    # If correction relates to a prompt issue, suggest prompt refinement
    if correction.type in ("missing_requirement", "wrong_pattern"):
        await suggest_prompt_refinement(
            module=correction.module,
            issue=correction.explanation,
            example_correction=correction.corrected
        )
    
    return signals
```

## Improvement Alerts

### Types
| Alert Type | Trigger | Suggested Action |
|------------|---------|-----------------|
| `quality_drop` | Avg quality < 85% of baseline | Review recent prompts, check for drift |
| `cost_spike` | Daily cost > 2x average | Check for loops, reduce context size |
| `success_rate_drop` | Agent success < 85% | Switch model, refine prompts |
| `recurring_failure` | Same failure > 3x in 7 days | Fix root cause in prompt/workflow |
| `estimation_drift` | Actual > 2x estimated consistently | Recalibrate estimation model |
| `unused_context` | Context files never referenced | Remove from loading formula |

### Alert Resolution Workflow
```
[Alert Created]
    ↓
[Notify relevant team lead]
    ↓
[Team lead investigates]
    ↓
[Resolution options]
    ├── Refine prompt → Update repo file → Version bump
    ├── Change agent config → Update agent_prompts table
    ├── Adjust policy → Update approval_policies
    ├── Retrain/switch model → Update agent config
    └── Accept as normal → Dismiss alert with reason
    ↓
[Record resolution in alert]
    ↓
[Monitor for recurrence]
```

## Prompt Refinement Suggestions

```python
async def suggest_prompt_refinement(module: str, issue: str, example_correction: str):
    """
    When corrections indicate a prompt issue, suggest specific refinements.
    """
    # Find the relevant prompt file
    repo = await get_active_repo_for_module(module)
    relevant_files = await find_related_files(repo, module, issue)
    
    suggestion = {
        "type": "prompt_refinement",
        "module": module,
        "issue_description": issue,
        "affected_files": relevant_files,
        "example": {
            "before": "AI generated this...",
            "after": "Human corrected to this...",
            "explanation": example_correction
        },
        "recommended_action": f"Add explicit instruction in {relevant_files[0]} to prevent: {issue}",
        "priority": calculate_priority(module, issue),
        "created_at": now()
    }
    
    await db.create_improvement_suggestion(suggestion)
    await notify_team_lead(module, suggestion)
```

## Metrics Dashboard Data

### Real-time Metrics (WebSocket)
```json
{
  "active_executions": 3,
  "steps_running": 7,
  "tokens_per_minute": 12500,
  "cost_today_usd": 8.30,
  "success_rate_24h": 0.92,
  "avg_step_duration_seconds": 45
}
```

### Historical Metrics (API)
```json
{
  "period": "last_30_days",
  "total_executions": 245,
  "total_cost_usd": 180.50,
  "avg_cost_per_execution_usd": 0.74,
  "success_rate": 0.91,
  "avg_quality_score": 8.2,
  "human_correction_rate": 0.15,
  "top_failure_types": [
    {"type": "compilation_error", "count": 12},
    {"type": "test_failure", "count": 8},
    {"type": "timeout", "count": 3}
  ],
  "cost_trend": "decreasing",  // improving | stable | decreasing
  "quality_trend": "improving"
}
```

## A/B Testing (Prompt Experiments)

```python
class PromptExperiment:
    """
    Run A/B tests on prompt variations to find optimal wording.
    """
    
    async def create_experiment(
        self,
        name: str,
        file_path: str,
        variant_a: str,  # Current content
        variant_b: str,  # New content
        sample_size: int = 20,
        success_metric: str = "quality_score"
    ):
        experiment = await db.create_experiment(
            name=name,
            file_path=file_path,
            variants={"a": variant_a, "b": variant_b},
            sample_size=sample_size,
            success_metric=success_metric,
            status="running"
        )
        return experiment
    
    async def assign_variant(self, experiment_id: str, execution_id: str) -> str:
        """Randomly assign variant, maintaining 50/50 split."""
        experiment = await db.get_experiment(experiment_id)
        counts = await db.get_variant_counts(experiment_id)
        
        # Balance assignment
        if counts["a"] <= counts["b"]:
            variant = "a"
        else:
            variant = "b"
        
        await db.assign_variant(experiment_id, execution_id, variant)
        return variant
    
    async def check_significance(self, experiment_id: str) -> dict:
        """Check if experiment has reached statistical significance."""
        results = await db.get_experiment_results(experiment_id)
        
        if results["total_samples"] < results["required_samples"]:
            return {"status": "insufficient_data"}
        
        # Mann-Whitney U test for quality scores
        from scipy.stats import mannwhitneyu
        stat, p_value = mannwhitneyu(results["a_scores"], results["b_scores"])
        
        if p_value < 0.05:
            winner = "b" if mean(results["b_scores"]) > mean(results["a_scores"]) else "a"
            return {
                "status": "significant",
                "winner": winner,
                "p_value": p_value,
                "improvement": abs(mean(results["b_scores"]) - mean(results["a_scores"]))
            }
        
        return {"status": "not_significant", "p_value": p_value}
```
