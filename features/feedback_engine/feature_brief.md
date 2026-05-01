# Feedback Engine — Feature Brief

## Module Identity

| Field | Value |
|-------|-------|
| Module ID | `feedback_engine` |
| Layer | 8 (Learning) |
| Owner | Backend Team Lead |
| Status | Planned |
| Priority | High (continuous improvement) |

## Purpose

The Feedback Engine closes the learning loop. It collects outcomes from every execution, human corrections, quality metrics, and cost data. It uses this information to improve future planning (better cost estimates, better risk assessment), improve agent performance (prompt tuning, pattern selection), and provide management dashboards.

## Core Capabilities

### 1. Outcome Collection
- Automatic collection after each execution completes
- Track: success/failure, tokens used, time taken, human corrections needed
- Compare estimated vs actual (cost, time, risk)
- Capture human edits to agent output (diff analysis)
- Track downstream effects (did the code work in production?)

### 2. Quality Scoring
- Per-execution quality score (0-10):
  - Code compiles: +2
  - Tests pass: +2
  - Review approved (no critical issues): +2
  - No human corrections needed: +2
  - Meets acceptance criteria: +2
- Per-agent quality tracking over time
- Per-module quality tracking
- Trend analysis (improving/degrading)

### 3. Prompt Optimization
- A/B testing of agent system prompts
- Track which prompt versions produce better outcomes
- Suggest prompt modifications based on failure patterns
- Automatic prompt versioning with rollback
- Correlation analysis: prompt change → quality change

### 4. Cost Analytics
- Real-time cost tracking per project, module, agent
- Budget vs actual comparison
- Cost optimization suggestions:
  - "Module X could use Sonnet instead of Opus (90% success rate)"
  - "Pattern Y reduces token usage by 30%"
- Forecasting: projected monthly cost based on velocity

### 5. Management Dashboard Data
- Project health overview
- Team velocity metrics
- Agent performance comparison
- Cost trends and projections
- Quality trends
- Bottleneck identification

### 6. Continuous Improvement Triggers
- When success rate drops below threshold → alert
- When cost exceeds budget → alert + suggest optimization
- When human correction rate increases → suggest prompt review
- When new pattern emerges → store in Knowledge Base
- When anti-pattern detected → add to review checklist

## API Endpoints

### POST /api/v1/feedback/record
Record execution outcome.
```json
// Request
{
  "execution_id": "exec_abc",
  "outcome": {
    "status": "success",
    "quality_score": 8,
    "quality_breakdown": {
      "compiles": true,
      "tests_pass": true,
      "review_approved": true,
      "no_human_corrections": false,
      "meets_criteria": true
    },
    "actual_tokens": 18500,
    "actual_cost_usd": 0.92,
    "actual_duration_minutes": 22,
    "human_corrections": [
      {"file": "src/routes/orders.ts", "type": "minor_fix", "description": "Fixed validation logic"}
    ]
  }
}

// Response 200
{
  "recorded": true,
  "insights": [
    "Human correction rate for code_agent increased 5% this week",
    "Estimated cost was 15% lower than actual — calibrating model"
  ]
}
```

### GET /api/v1/feedback/dashboard
Get dashboard metrics.
```json
// Response 200
{
  "overview": {
    "total_executions": 156,
    "success_rate": 0.91,
    "avg_quality_score": 7.8,
    "total_cost_usd": 234.50,
    "total_tokens": 2450000
  },
  "trends": {
    "quality_7d": [7.5, 7.6, 7.8, 7.9, 7.8, 8.0, 7.8],
    "cost_7d": [32.10, 28.50, 35.20, 30.00, 33.40, 38.00, 37.30],
    "velocity_7d": [12, 15, 14, 18, 16, 20, 21]
  },
  "agents": {
    "code_agent": {"success_rate": 0.88, "avg_quality": 7.5, "avg_tokens": 5200},
    "test_agent": {"success_rate": 0.94, "avg_quality": 8.2, "avg_tokens": 3100},
    "review_agent": {"success_rate": 0.97, "avg_quality": 8.8, "avg_tokens": 2800}
  },
  "alerts": [
    {"type": "quality_drop", "module": "payments", "message": "Quality dropped 15% this week"},
    {"type": "cost_spike", "module": "reporting", "message": "Cost 2x above average"}
  ]
}
```

### GET /api/v1/feedback/agent-performance/:agentType
Get detailed agent performance metrics.

### GET /api/v1/feedback/cost-analysis
Get cost breakdown and optimization suggestions.
```json
{
  "monthly_cost": 890.50,
  "projected_monthly": 1050.00,
  "budget": 1200.00,
  "breakdown_by_agent": {
    "code_agent": 520.00,
    "test_agent": 180.00,
    "review_agent": 120.00,
    "docs_agent": 40.00,
    "devops_agent": 30.50
  },
  "optimization_suggestions": [
    {
      "suggestion": "Use Sonnet for test_agent (current success rate with Sonnet: 92%)",
      "estimated_savings_monthly": 90.00,
      "risk": "low"
    },
    {
      "suggestion": "Enable pattern reuse for common migrations",
      "estimated_savings_monthly": 60.00,
      "risk": "low"
    }
  ]
}
```

### GET /api/v1/feedback/prompt-performance
Get prompt version performance comparison.

### POST /api/v1/feedback/prompt-experiment
Start a prompt A/B test.
```json
{
  "agent_type": "code_agent",
  "variant_a": {"prompt_version": 3, "description": "Current production"},
  "variant_b": {"prompt_version": 4, "description": "Added explicit error handling instruction"},
  "traffic_split": 0.5,
  "min_samples": 20,
  "success_metric": "quality_score"
}
```

## Data Model

### Table: execution_outcomes
```sql
CREATE TABLE execution_outcomes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  execution_id UUID NOT NULL REFERENCES executions(id) UNIQUE,
  project_id UUID NOT NULL REFERENCES projects(id),
  
  status VARCHAR(20) NOT NULL,
  quality_score DECIMAL(3,1) NOT NULL,
  quality_breakdown JSONB NOT NULL,
  
  estimated_tokens INTEGER NOT NULL,
  actual_tokens INTEGER NOT NULL,
  estimated_cost_usd DECIMAL(10,4) NOT NULL,
  actual_cost_usd DECIMAL(10,4) NOT NULL,
  estimated_duration_minutes INTEGER NOT NULL,
  actual_duration_minutes INTEGER NOT NULL,
  
  human_corrections JSONB DEFAULT '[]',
  human_correction_count INTEGER NOT NULL DEFAULT 0,
  
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_outcomes_project ON execution_outcomes(project_id);
CREATE INDEX idx_outcomes_quality ON execution_outcomes(quality_score);
CREATE INDEX idx_outcomes_created ON execution_outcomes(created_at DESC);
```

### Table: agent_metrics (aggregated daily)
```sql
CREATE TABLE agent_metrics (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  date DATE NOT NULL,
  agent_type VARCHAR(30) NOT NULL,
  project_id UUID REFERENCES projects(id),
  
  executions_count INTEGER NOT NULL DEFAULT 0,
  success_count INTEGER NOT NULL DEFAULT 0,
  failure_count INTEGER NOT NULL DEFAULT 0,
  
  avg_quality_score DECIMAL(3,1),
  avg_tokens INTEGER,
  avg_cost_usd DECIMAL(10,4),
  avg_duration_seconds INTEGER,
  
  human_correction_rate DECIMAL(3,2),
  
  UNIQUE(date, agent_type, project_id)
);

CREATE INDEX idx_agent_metrics_date ON agent_metrics(date DESC);
```

### Table: prompt_experiments
```sql
CREATE TABLE prompt_experiments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  agent_type VARCHAR(30) NOT NULL,
  
  status VARCHAR(20) NOT NULL DEFAULT 'running',
  -- Status: running | completed | cancelled
  
  variant_a JSONB NOT NULL,
  variant_b JSONB NOT NULL,
  traffic_split DECIMAL(3,2) NOT NULL DEFAULT 0.5,
  
  results_a JSONB DEFAULT '{}',
  results_b JSONB DEFAULT '{}',
  
  winner VARCHAR(10),  -- 'a' | 'b' | 'inconclusive'
  confidence DECIMAL(3,2),
  
  min_samples INTEGER NOT NULL DEFAULT 20,
  current_samples_a INTEGER NOT NULL DEFAULT 0,
  current_samples_b INTEGER NOT NULL DEFAULT 0,
  
  started_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  completed_at TIMESTAMP WITH TIME ZONE
);
```

### Table: improvement_alerts
```sql
CREATE TABLE improvement_alerts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID REFERENCES projects(id),
  
  type VARCHAR(30) NOT NULL,
  -- Types: quality_drop | cost_spike | success_rate_drop | pattern_detected | anti_pattern
  
  severity VARCHAR(10) NOT NULL,
  -- Severity: info | warning | critical
  
  message TEXT NOT NULL,
  details JSONB DEFAULT '{}',
  
  acknowledged BOOLEAN NOT NULL DEFAULT FALSE,
  acknowledged_by UUID REFERENCES users(id),
  
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_alerts_project ON improvement_alerts(project_id);
CREATE INDEX idx_alerts_type ON improvement_alerts(type);
CREATE INDEX idx_alerts_acknowledged ON improvement_alerts(acknowledged);
```

## Workflow: Feedback Collection Cycle

```
Execution Completed
    ↓
[Collect Raw Metrics]
  - tokens, cost, duration, status
    ↓
[Calculate Quality Score]
  - compile check, test results, review status
    ↓
[Compare Estimated vs Actual]
  - cost accuracy, time accuracy
    ↓
[Detect Human Corrections]
  - diff between agent output and final merged code
    ↓
[Store Outcome]
    ↓
[Update Aggregated Metrics]
    ↓
[Check Alert Thresholds]
    ├── Threshold breached → Generate Alert
    └── Normal → Continue
    ↓
[Update Knowledge Base]
  - Store successful patterns
  - Flag anti-patterns
    ↓
[Update Planning Calibration]
  - Adjust cost estimation model
  - Adjust risk scoring weights
    ↓
[Check Prompt Experiments]
  - If experiment running, record sample
  - If min_samples reached, evaluate winner
```

## Configuration

```yaml
feedback_engine:
  collection:
    auto_collect: true
    collect_delay_seconds: 10  # Wait for final state
  
  quality:
    min_acceptable_score: 6.0
    alert_on_drop_percent: 15
    scoring_weights:
      compiles: 2
      tests_pass: 2
      review_approved: 2
      no_corrections: 2
      meets_criteria: 2
  
  alerts:
    quality_drop_window_days: 7
    cost_spike_multiplier: 2.0
    success_rate_min: 0.85
    check_interval_minutes: 60
  
  experiments:
    min_samples_default: 20
    confidence_threshold: 0.95
    max_experiment_duration_days: 14
  
  aggregation:
    daily_rollup_time: "02:00"  # 2 AM
    retention_days: 365
```

## Success Metrics

| Metric | Target |
|--------|--------|
| Data collection coverage | 100% of executions |
| Alert detection latency | < 1 hour |
| Cost estimation accuracy improvement | 10% per month |
| Quality trend visibility | Real-time (< 5 min delay) |
| Prompt experiment turnaround | < 7 days |
| Pattern reuse increase | 5% per month |
