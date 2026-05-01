# AI Software Factory Platform — Prompt Repository

## Overview

This repository is the **single source of truth** for building the AI Software Factory Platform — a hybrid human-AI system that produces software by generating structured prompts (not code directly). The platform's core innovation:

> **Goal → Plan → Prompt → Code** (NOT Goal → Code)

The platform does not produce code. It produces **prompt structures that generate code**.

## Architecture (8 Layers)

| Layer | Name | Responsibility | Actor |
|-------|------|---------------|-------|
| 1 | Command UI | Task submission, plan visualization, approval | Human |
| 2 | Planning Service | Goal → Plan Schema (tasks, deps, agents, risk) | AI Agent |
| 3 | Approval Gateway | Scope/Cost/Risk review → Approve/Modify/Reject | Human |
| 4 | Prompt Factory ⭐ | Plan Schema → Prompt Repository structure | AI + Human |
| 5 | Orchestrator | DAG construction, parallel execution, state mgmt | Engine |
| 6 | Agent Runtime | Consume prompts, produce code/test/docs | AI Agents |
| 7 | Knowledge Base | RAG, codebase embeddings, project memory | System |
| 8 | Feedback Engine | Success rate, human edits, bug rate collection | System |

## Core Flow (Iterative)

Every stage operates in a **loop until finalized** — not single-pass:

```
[Human] Submit Goal/PRD/Feature
    ↕ (iterate until approved)
[Planning Service] → Plan Schema JSON
    ↕ (iterate until approved)
[Approval Gateway] → Human approves
    ↕ (iterate until approved)
[Prompt Factory] → Update Repository (version bump)
    ↕ (iterate until approved)
[Orchestrator] → DAG → Agents activated
    ↓
[Agent Runtime] → Code/Test/Review/DevOps
    ↕ (iterate until validated)
[Validation] → Human + AI verify
    ↓
[Feedback Loop] → Data returns to Planner + Prompt Factory
    ↓
[Team Lead] → Merge approved output into Repository
```

**Key Rule:** After approval at each stage, the team lead merges the output into the relevant section of this repository and bumps the version.

## Repository Structure

```
ai_factory_prompt_repo/
├── README.md                          ← This file
├── shared/
│   ├── backend.md                     ← Server architecture, API standards, tech stack
│   ├── frontend.md                    ← Client architecture, component patterns
│   ├── design.md                      ← Design system, theming, accessibility
│   ├── app.md                         ← Platform configuration, environments
│   ├── personas.md                    ← User roles, permissions matrix
│   └── versioning.md                  ← Version control rules, SemVer, migrations
├── features/
│   ├── command_ui/                    ← Layer 1: Human interface
│   ├── planning_service/              ← Layer 2: AI planning engine
│   ├── approval_gateway/              ← Layer 3: Human approval workflow
│   ├── prompt_factory/                ← Layer 4: Core differentiator
│   ├── orchestrator/                  ← Layer 5: Execution coordination
│   ├── agent_runtime/                 ← Layer 6: Agent execution environment
│   ├── knowledge_base/                ← Layer 7: RAG + memory
│   └── feedback_engine/               ← Layer 8: Learning loop
├── context_packs/
│   ├── backend/                       ← Server-side patterns and conventions
│   ├── frontend/                      ← Client-side patterns and conventions
│   ├── command_room/                  ← Architecture overview, dependencies
│   ├── memory/                        ← Decisions log, known issues, glossary
│   ├── product_management/            ← User stories, acceptance criteria
│   └── operations/                    ← Deployment, monitoring, scaling
└── changes/
    ├── manifest.yaml                  ← Version registry
    ├── VERSIONING_GUIDE.md            ← Rules for version bumps
    └── entries/                        ← Changelog per version
```

## How AI Consumes This Repository

1. AI receives a task (e.g., "implement planning service API")
2. AI loads ONLY 3-5 relevant files (guided by context table in shared/app.md)
3. AI produces output (code, tests, docs)
4. Team lead reviews and merges into repository
5. Version is bumped, changelog entry written

**Rule 7:** AI only modifies loaded files — never guesses about other files.

## Tech Stack

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Command UI | Next.js 14 + TypeScript | SSR, App Router, React Server Components |
| Planning Service | Python FastAPI + LangChain | LLM integration, async, typed |
| Prompt Factory | Python FastAPI | Core logic, template engine |
| Orchestrator | LangGraph + Redis | DAG, state management, queuing |
| Agent Runtime | Docker containers + Claude API | Isolation, scalability |
| Knowledge Base | PostgreSQL + pgvector + ChromaDB | Hybrid search, embeddings |
| Feedback Engine | Python + ClickHouse | Analytics, time-series |
| Infrastructure | Kubernetes + Helm | Orchestration, scaling |
| CI/CD | GitHub Actions + ArgoCD | GitOps, automated deployment |
| Monitoring | Prometheus + Grafana | Observability |

## Version

Current: **v0.1.0** (Initial repository structure)
