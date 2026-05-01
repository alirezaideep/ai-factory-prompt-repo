# Glossary

## Core Concepts

| Term | Definition |
|------|-----------|
| **AI Factory** | The complete platform for AI-assisted software development lifecycle |
| **Prompt Repository** | Git-based structured Markdown repo containing all project knowledge |
| **Plan Schema** | JSON structure defining execution steps, dependencies, agents, and budgets |
| **Execution** | A single run of a plan through the orchestrator |
| **Step** | Atomic unit of work within an execution, assigned to one agent |
| **Agent** | AI model instance configured for a specific task type |
| **Loading Formula** | Rule defining which files to load for each task type |
| **Context Pack** | Collection of related conventions/patterns for a specific domain |
| **Feature Brief** | High-level description of a module's purpose, scope, and interfaces |
| **Delta** | Incremental change to the prompt repository (vs. full rewrite) |

## Architecture Terms

| Term | Definition |
|------|-----------|
| **Command Room** | Human-facing UI for task creation, monitoring, and approval |
| **Planning Service** | AI service that decomposes goals into executable plans |
| **Approval Gateway** | Human checkpoint before execution begins |
| **Prompt Factory** | Service that builds/updates the prompt repository |
| **Orchestrator** | Engine that executes plans as DAGs with state management |
| **Agent Runtime** | Environment where AI agents execute individual steps |
| **Knowledge Base** | Vector DB + embeddings for semantic code search |
| **Feedback Engine** | System that collects signals and improves future performance |

## Execution Terms

| Term | Definition |
|------|-----------|
| **DAG** | Directed Acyclic Graph — execution dependency structure |
| **Fan-out** | One step producing multiple parallel downstream steps |
| **Fan-in** | Multiple steps converging into one downstream step |
| **Checkpoint** | Saved execution state for resume after failure |
| **DLQ** | Dead Letter Queue — failed steps awaiting manual review |
| **Circuit Breaker** | Pattern to prevent cascading failures from external services |
| **Self-fix Loop** | Agent attempting to fix its own errors iteratively |
| **Escalation** | Pausing execution to request human intervention |

## Versioning Terms

| Term | Definition |
|------|-----------|
| **SemVer** | Semantic Versioning (MAJOR.MINOR.PATCH) |
| **Breaking Change** | Change requiring consumers to update their code |
| **Migration** | Script to transform data/schema from one version to another |
| **Changelog Entry** | Structured record of what changed in a version |
| **Baseline** | Full snapshot of repository at a specific version |
| **Delta Update** | Only the changed files/sections since last version |

## Role Terms

| Term | Definition |
|------|-----------|
| **Admin** | Full system access, manages users and global settings |
| **Team Lead** | Manages team projects, approves plans and merges |
| **Developer** | Creates tasks, reviews AI output, provides feedback |
| **Viewer** | Read-only access to projects and executions |
| **Agent Owner** | Person responsible for an agent type's configuration |

## Abbreviations

| Abbreviation | Full Form |
|-------------|-----------|
| ADR | Architecture Decision Record |
| RBAC | Role-Based Access Control |
| ABAC | Attribute-Based Access Control |
| SSE | Server-Sent Events |
| WS | WebSocket |
| RAG | Retrieval-Augmented Generation |
| LLM | Large Language Model |
| DAG | Directed Acyclic Graph |
| DLQ | Dead Letter Queue |
| CI/CD | Continuous Integration / Continuous Deployment |
| ORM | Object-Relational Mapping |
| DTO | Data Transfer Object |
