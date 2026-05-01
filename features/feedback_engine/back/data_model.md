# Feedback Engine — Data Model

## Note
Primary tables (execution_outcomes, agent_metrics, prompt_experiments, improvement_alerts) are defined in feature_brief.md. This file covers analytics views and aggregation logic.

## Materialized Views

### Daily Project Summary
```sql
CREATE MATERIALIZED VIEW mv_daily_project_summary AS
SELECT
  date_trunc('day', eo.created_at)::DATE as date,
  eo.project_id,
  COUNT(*) as total_executions,
  COUNT(*) FILTER (WHERE eo.status = 'success') as successful,
  COUNT(*) FILTER (WHERE eo.status = 'failed') as failed,
  AVG(eo.quality_score) as avg_quality,
  SUM(eo.actual_tokens) as total_tokens,
  SUM(eo.actual_cost_usd) as total_cost,
  AVG(eo.actual_duration_minutes) as avg_duration,
  AVG(eo.human_correction_count) as avg_corrections
FROM execution_outcomes eo
GROUP BY 1, 2;

-- Refresh daily at 2 AM
CREATE INDEX idx_mv_daily_project ON mv_daily_project_summary(date DESC, project_id);
```

### Agent Performance Leaderboard
```sql
CREATE MATERIALIZED VIEW mv_agent_performance AS
SELECT
  ae.agent_type,
  ae.model,
  COUNT(*) as total_executions,
  COUNT(*) FILTER (WHERE ae.status = 'completed') as successful,
  ROUND(COUNT(*) FILTER (WHERE ae.status = 'completed')::DECIMAL / NULLIF(COUNT(*), 0), 3) as success_rate,
  AVG(ae.total_tokens) as avg_tokens,
  AVG(ae.cost_usd) as avg_cost,
  AVG(ae.duration_ms) as avg_duration_ms,
  PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY ae.duration_ms) as p95_duration_ms
FROM agent_executions ae
WHERE ae.created_at > NOW() - INTERVAL '30 days'
GROUP BY ae.agent_type, ae.model;
```

### Cost Forecast
```sql
CREATE MATERIALIZED VIEW mv_cost_forecast AS
SELECT
  project_id,
  date_trunc('week', created_at)::DATE as week,
  SUM(actual_cost_usd) as weekly_cost,
  SUM(actual_tokens) as weekly_tokens,
  COUNT(*) as weekly_executions,
  AVG(actual_cost_usd) as avg_cost_per_execution
FROM execution_outcomes
WHERE created_at > NOW() - INTERVAL '90 days'
GROUP BY project_id, date_trunc('week', created_at);
```

## Aggregation Jobs

### Daily Rollup (runs at 2 AM)
```python
async def daily_rollup():
    """Aggregate yesterday's data into agent_metrics table."""
    yesterday = date.today() - timedelta(days=1)
    
    # Get all agent executions from yesterday
    executions = await db.query("""
        SELECT agent_type, project_id, status, total_tokens, cost_usd, duration_ms
        FROM agent_executions
        WHERE DATE(created_at) = $1
    """, yesterday)
    
    # Group by agent_type + project_id
    groups = group_by(executions, ["agent_type", "project_id"])
    
    for key, group in groups.items():
        metrics = {
            "date": yesterday,
            "agent_type": key.agent_type,
            "project_id": key.project_id,
            "executions_count": len(group),
            "success_count": sum(1 for e in group if e.status == "completed"),
            "failure_count": sum(1 for e in group if e.status == "failed"),
            "avg_quality_score": calculate_avg_quality(group),
            "avg_tokens": mean([e.total_tokens for e in group]),
            "avg_cost_usd": mean([e.cost_usd for e in group]),
            "avg_duration_seconds": mean([e.duration_ms / 1000 for e in group]),
            "human_correction_rate": calculate_correction_rate(group)
        }
        
        await db.upsert_agent_metrics(metrics)
    
    # Refresh materialized views
    await db.execute("REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_project_summary")
    await db.execute("REFRESH MATERIALIZED VIEW CONCURRENTLY mv_agent_performance")
    await db.execute("REFRESH MATERIALIZED VIEW CONCURRENTLY mv_cost_forecast")
```

### Alert Check (runs every hour)
```python
async def check_alerts():
    """Check for quality drops, cost spikes, and other anomalies."""
    
    # Quality drop check
    for project_id in await get_active_projects():
        recent_quality = await get_avg_quality(project_id, days=7)
        baseline_quality = await get_avg_quality(project_id, days=30)
        
        if baseline_quality > 0 and recent_quality < baseline_quality * 0.85:
            await create_alert(
                project_id=project_id,
                type="quality_drop",
                severity="warning",
                message=f"Quality dropped {((baseline_quality - recent_quality) / baseline_quality * 100):.0f}% in last 7 days",
                details={"recent": recent_quality, "baseline": baseline_quality}
            )
    
    # Cost spike check
    for project_id in await get_active_projects():
        today_cost = await get_daily_cost(project_id, date.today())
        avg_daily_cost = await get_avg_daily_cost(project_id, days=14)
        
        if avg_daily_cost > 0 and today_cost > avg_daily_cost * 2.0:
            await create_alert(
                project_id=project_id,
                type="cost_spike",
                severity="warning",
                message=f"Today's cost (${today_cost:.2f}) is {today_cost/avg_daily_cost:.1f}x above average",
                details={"today": today_cost, "average": avg_daily_cost}
            )
    
    # Success rate check
    for agent_type in AGENT_TYPES:
        recent_rate = await get_success_rate(agent_type, days=3)
        if recent_rate < 0.85:
            await create_alert(
                type="success_rate_drop",
                severity="critical" if recent_rate < 0.7 else "warning",
                message=f"{agent_type} success rate dropped to {recent_rate*100:.0f}%"
            )
```

## Calibration Model

```python
class CostEstimationCalibrator:
    """
    Continuously calibrate cost/time estimates based on actual outcomes.
    Uses exponential moving average with decay factor.
    """
    
    DECAY = 0.1  # Weight of new observation
    
    async def update_estimates(self, execution_outcome):
        """Update estimation model with new data point."""
        
        # Get current calibration factors
        factors = await db.get_calibration_factors(
            agent_type=execution_outcome.agent_type,
            task_complexity=execution_outcome.complexity
        )
        
        # Update cost factor
        if execution_outcome.estimated_cost > 0:
            actual_ratio = execution_outcome.actual_cost / execution_outcome.estimated_cost
            new_cost_factor = (1 - self.DECAY) * factors.cost_factor + self.DECAY * actual_ratio
        
        # Update time factor
        if execution_outcome.estimated_duration > 0:
            actual_ratio = execution_outcome.actual_duration / execution_outcome.estimated_duration
            new_time_factor = (1 - self.DECAY) * factors.time_factor + self.DECAY * actual_ratio
        
        # Save updated factors
        await db.update_calibration_factors(
            agent_type=execution_outcome.agent_type,
            task_complexity=execution_outcome.complexity,
            cost_factor=new_cost_factor,
            time_factor=new_time_factor
        )
```

## Table: calibration_factors
```sql
CREATE TABLE calibration_factors (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  agent_type VARCHAR(30) NOT NULL,
  task_complexity VARCHAR(10) NOT NULL,  -- low | medium | high
  
  cost_factor DECIMAL(5,3) NOT NULL DEFAULT 1.0,
  time_factor DECIMAL(5,3) NOT NULL DEFAULT 1.0,
  token_factor DECIMAL(5,3) NOT NULL DEFAULT 1.0,
  
  sample_count INTEGER NOT NULL DEFAULT 0,
  last_updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  
  UNIQUE(agent_type, task_complexity)
);
```
