# Feedback Engine — API Endpoints

## Base Path: `/api/v1/feedback`

### Signal Collection

#### POST /api/v1/feedback/signals
Record a feedback signal from any source.

**Request:**
```json
{
  "type": "execution_outcome",
  "source": "orchestrator",
  "executionId": "exec-uuid",
  "stepId": "step-uuid",
  "agentType": "code_generation",
  "signal": {
    "outcome": "success",
    "duration": 45,
    "tokensUsed": 42000,
    "selfFixAttempts": 1,
    "humanEditsAfter": 3,
    "qualityScore": 0.85
  }
}
```

Signal types:
- `execution_outcome` — Step success/failure metrics
- `human_edit` — Human modified AI output (diff tracked)
- `approval_decision` — Approval accepted/rejected/modified
- `bug_report` — Bug found in AI-generated code
- `user_rating` — Explicit quality rating from user

#### GET /api/v1/feedback/signals
Query signals with filtering for analysis.

**Query Params:** `?type=human_edit&agent=code-gen&from=2025-01-01&limit=100`

### Analytics

#### GET /api/v1/feedback/analytics/agents
Get aggregated performance analytics per agent type.

**Response:**
```json
{
  "agents": [
    {
      "agentType": "code_generation",
      "period": "last_30_days",
      "successRate": 0.94,
      "avgQualityScore": 0.82,
      "humanEditRate": 0.35,
      "avgEditsPerOutput": 2.1,
      "commonIssues": [
        { "pattern": "missing_error_handling", "frequency": 0.15 },
        { "pattern": "incomplete_types", "frequency": 0.12 }
      ],
      "trend": "improving"
    }
  ]
}
```

#### GET /api/v1/feedback/analytics/prompts
Get prompt effectiveness analytics.

#### GET /api/v1/feedback/analytics/cost
Get cost analytics with breakdown by project, agent, model.

### Improvement Suggestions

#### GET /api/v1/feedback/suggestions
Get AI-generated improvement suggestions based on collected signals.

**Response:**
```json
{
  "suggestions": [
    {
      "id": "sug-uuid",
      "type": "prompt_improvement",
      "target": "agent:code-gen-v2",
      "confidence": 0.78,
      "description": "Add explicit error handling instruction to system prompt",
      "evidence": "15% of outputs lack try/catch blocks, human adds them 89% of the time",
      "suggestedChange": "Add to system prompt: 'Always wrap async operations in try/catch...'",
      "estimatedImpact": "+8% quality score"
    }
  ]
}
```

#### POST /api/v1/feedback/suggestions/{id}/apply
Apply a suggestion (with approval workflow).
