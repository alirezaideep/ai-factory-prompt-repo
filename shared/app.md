# Application Configuration

## Project Identity

| Field | Value |
|-------|-------|
| Name | AI Software Factory |
| Codename | `aisfactory` |
| Type | Internal Platform (B2B SaaS-ready) |
| Primary Language | English (code + docs) |
| UI Language | English (Persian RTL ready) |
| License | Proprietary (open-source components listed) |

## Environments

| Environment | Purpose | URL Pattern | Database |
|-------------|---------|-------------|----------|
| Local | Development | `localhost:3000` | Local PostgreSQL |
| Dev | Integration testing | `dev.factory.internal` | Shared dev DB |
| Staging | Pre-production | `staging.factory.internal` | Staging DB (prod clone) |
| Production | Live system | `factory.internal` | Production DB |

## Environment Variables

### Required (all services)
```
DATABASE_URL=postgresql://...
REDIS_URL=redis://...
JWT_SECRET=...
CORS_ORIGINS=...
LOG_LEVEL=info
```

### Planning Service
```
LLM_PROVIDER=claude
CLAUDE_API_KEY=...
CLAUDE_MODEL=claude-sonnet-4-20250514
FALLBACK_LLM_PROVIDER=openai
OPENAI_API_KEY=...
MAX_TOKENS_PER_PLAN=8000
PLANNING_TIMEOUT_SECONDS=120
```

### Prompt Factory
```
REPO_BASE_PATH=/data/prompt-repos
GIT_AUTHOR_NAME=AI Factory Bot
GIT_AUTHOR_EMAIL=bot@factory.internal
TEMPLATE_VERSION=v1
```

### Orchestrator
```
MAX_PARALLEL_AGENTS=5
EXECUTION_TIMEOUT_SECONDS=600
RETRY_MAX_ATTEMPTS=3
DEAD_LETTER_QUEUE=dlq:executions
```

### Agent Runtime
```
AGENT_DOCKER_REGISTRY=registry.factory.internal
AGENT_MEMORY_LIMIT=2Gi
AGENT_CPU_LIMIT=2
SANDBOX_ENABLED=true
```

## Context Loading Table

**This is the most important table in the repository.** It tells AI which files to load for each type of task.

| Task Type | Required Files | Optional Files |
|-----------|---------------|----------------|
| **New feature (full)** | shared/backend.md, shared/frontend.md, shared/design.md, shared/versioning.md | context_packs/command_room/architecture_overview.md |
| **Backend API endpoint** | shared/backend.md, features/{module}/back/api_endpoints.md, features/{module}/back/data_model.md | context_packs/backend/api_conventions.md |
| **Frontend page** | shared/frontend.md, shared/design.md, features/{module}/front/pages.md, features/{module}/ui/wireframes.md | context_packs/frontend/component_patterns.md |
| **Database migration** | shared/backend.md, features/{module}/back/data_model.md, shared/versioning.md | context_packs/backend/database_patterns.md |
| **Workflow change** | features/{module}/workflows/*.md, features/{module}/feature_brief.md | context_packs/command_room/module_dependencies.md |
| **Bug fix (backend)** | features/{module}/back/service_logic.md, context_packs/memory/known_issues.md | features/{module}/back/api_endpoints.md |
| **Bug fix (frontend)** | features/{module}/front/pages.md, features/{module}/front/components.md | context_packs/memory/known_issues.md |
| **New agent type** | features/agent_runtime/feature_brief.md, features/agent_runtime/back/service_logic.md, shared/backend.md | features/orchestrator/workflows/dag_execution.md |
| **Prompt template** | features/prompt_factory/back/service_logic.md, features/prompt_factory/workflows/generation_flow.md | context_packs/product_management/acceptance_criteria.md |
| **DevOps/Deploy** | context_packs/operations/deployment_strategy.md, context_packs/operations/monitoring.md | shared/app.md |
| **Security fix** | context_packs/backend/auth_and_security.md, shared/backend.md | features/{module}/back/api_endpoints.md |
| **UI redesign** | shared/design.md, features/{module}/ui/wireframes.md, features/{module}/front/pages.md | shared/frontend.md |
| **Performance optimization** | context_packs/operations/monitoring.md, features/{module}/back/service_logic.md | shared/backend.md |
| **Version bump** | shared/versioning.md, changes/manifest.yaml | changes/VERSIONING_GUIDE.md |

## Feature Flags

| Flag | Default | Description |
|------|---------|-------------|
| `FF_PARALLEL_AGENTS` | true | Enable parallel agent execution |
| `FF_AUTO_APPROVE_LOW_RISK` | false | Auto-approve plans with risk < 0.3 |
| `FF_STREAMING_OUTPUT` | true | Stream agent output in real-time |
| `FF_FEEDBACK_COLLECTION` | true | Collect execution feedback |
| `FF_MULTI_PROJECT` | false | Support multiple projects |
| `FF_CUSTOM_AGENTS` | false | Allow user-defined agents |
| `FF_COST_TRACKING` | true | Track LLM token costs |

## External Integrations

| Service | Purpose | Auth Method |
|---------|---------|-------------|
| Claude API | Primary LLM (code generation) | API Key |
| OpenAI API | Fallback LLM | API Key |
| GitHub | Code repository, PR creation | OAuth App |
| GitLab | Alternative code repository | OAuth App |
| Slack | Notifications, approvals | Bot Token |
| Jira | Task import/sync | API Token |
| Linear | Task import/sync | API Key |
| Docker Registry | Agent images | Token |
| S3/MinIO | File storage | Access Key |

## Deployment Configuration

### Resource Requirements (per service)
| Service | CPU | Memory | Replicas (prod) |
|---------|-----|--------|-----------------|
| Command UI | 0.5 | 512Mi | 2 |
| Planning Service | 1 | 1Gi | 2 |
| Prompt Factory | 1 | 1Gi | 2 |
| Orchestrator | 2 | 2Gi | 3 |
| Agent Runtime | 2 | 2Gi | 5 (auto-scale) |
| Knowledge Base | 1 | 2Gi | 2 |
| Feedback Engine | 0.5 | 512Mi | 1 |
| PostgreSQL | 4 | 8Gi | 1 (HA cluster) |
| Redis | 1 | 2Gi | 3 (sentinel) |
| ClickHouse | 2 | 4Gi | 1 |
