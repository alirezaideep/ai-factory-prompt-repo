# Architecture Decision Log

## Format
Each decision follows ADR (Architecture Decision Record) format:
- **ID:** Sequential number
- **Date:** When decided
- **Status:** Proposed | Accepted | Deprecated | Superseded
- **Context:** Why this decision was needed
- **Decision:** What was decided
- **Consequences:** Positive and negative impacts
- **Alternatives Considered:** What else was evaluated

---

## ADR-001: Prompt Repository as Central Artifact

**Date:** 2025-01-15  
**Status:** Accepted

**Context:** Need a single source of truth that both humans and AI agents can read/write. Must support versioning, partial updates, and context-aware loading.

**Decision:** Use a Git-based Markdown repository as the central artifact. All project knowledge lives in structured `.md` files organized by function (shared, features, context_packs, changes).

**Consequences:**
- (+) Human-readable, version-controlled, diff-friendly
- (+) Any AI model can consume Markdown without special tooling
- (+) Git provides full history, branching, merging
- (-) No structured query capability (must use file search)
- (-) Large repos may exceed context windows (mitigated by loading formula)

**Alternatives Considered:**
- JSON/YAML structured database → Less readable, harder to edit manually
- Vector database only → Not human-readable, hard to version
- Wiki system → No programmatic access, hard to automate

---

## ADR-002: LangGraph for Orchestration

**Date:** 2025-01-20  
**Status:** Accepted

**Context:** Need a framework for DAG execution that supports parallel steps, conditional branching, human-in-the-loop, and state persistence.

**Decision:** Use LangGraph (LangChain ecosystem) as the orchestration engine.

**Consequences:**
- (+) Built-in state management, checkpointing, human-in-the-loop
- (+) Native LLM integration, tool calling support
- (+) Active community, good documentation
- (-) Python-only (backend must be Python or hybrid)
- (-) Relatively new, may have breaking changes

**Alternatives Considered:**
- Temporal.io → More mature but heavier, Java/Go focused
- Prefect/Dagster → Data pipeline focused, not AI-native
- Custom implementation → Full control but high maintenance cost

---

## ADR-003: Iterative Approval Model

**Date:** 2025-01-22  
**Status:** Accepted

**Context:** AI output quality varies. Need human oversight without creating bottlenecks.

**Decision:** Every phase operates in a loop (AI produces → Human reviews → AI revises → ... → Human approves). Team leads merge approved output into the repository.

**Consequences:**
- (+) Quality control at every stage
- (+) Humans stay in control without doing the work
- (+) Learning opportunity (feedback improves future output)
- (-) Slower than fully autonomous execution
- (-) Requires available reviewers (can bottleneck)

**Alternatives Considered:**
- Fully autonomous (no review) → Too risky for production code
- Review only at end → Expensive to fix issues found late
- Automated review only → Not reliable enough yet

---

## ADR-004: Hybrid Model Strategy

**Date:** 2025-02-01  
**Status:** Accepted

**Context:** Different tasks need different capabilities. Cost optimization requires using cheaper models where possible.

**Decision:** Use Claude Opus for complex reasoning (planning, architecture), Claude Sonnet for code generation, and cheaper models (Haiku, GPT-4o-mini) for simple tasks. Implement automatic fallback chain.

**Consequences:**
- (+) Cost optimization (60-70% cheaper than all-Opus)
- (+) Resilience (fallback if primary model unavailable)
- (-) Inconsistency between models (different coding styles)
- (-) Complexity in model selection logic

---

## ADR-005: Monorepo with Service Boundaries

**Date:** 2025-02-05  
**Status:** Accepted

**Context:** Multiple services (Command UI, Planning, Orchestrator, etc.) need to be developed. Need to balance independence with code sharing.

**Decision:** Monorepo using Turborepo/Nx with clear package boundaries. Shared types in `packages/shared`, each service in `apps/`.

**Consequences:**
- (+) Shared types prevent drift
- (+) Atomic commits across services
- (+) Single CI pipeline
- (-) Larger repo size
- (-) Build complexity

---

## Template for New Decisions

```markdown
## ADR-XXX: [Title]

**Date:** YYYY-MM-DD  
**Status:** Proposed | Accepted | Deprecated | Superseded

**Context:** [Why is this decision needed?]

**Decision:** [What was decided]

**Consequences:**
- (+) [Positive impact]
- (-) [Negative impact]

**Alternatives Considered:**
- [Alternative 1] → [Why rejected]
- [Alternative 2] → [Why rejected]
```
