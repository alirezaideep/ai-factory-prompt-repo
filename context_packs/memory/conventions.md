# Project Conventions

## ⚠️ CRITICAL: Prompt Repository as Single Source of Truth

**This repository IS the project.** Every change to the product MUST be reflected here FIRST.

### Golden Rule
> The Prompt Repository at `https://github.com/alirezaideep/ai-factory-prompt-repo` is the
> single source of truth for all project decisions, architecture, data models, APIs, workflows,
> and UI specifications. No code is produced without first updating the relevant files in this repo.

### Update Protocol
1. **Before any code change:** Update the relevant `.md` file(s) in this repo
2. **After any code change:** Commit the repo update with proper version bump
3. **Every session:** AI must read the latest state of relevant files before producing output
4. **Conflict resolution:** If code and repo disagree, THE REPO WINS

### What Gets Updated
| Change Type | Files to Update | Version Bump |
|-------------|----------------|---------------|
| New feature | `features/<name>/feature_brief.md` + subdirs | MINOR |
| API change | `features/<name>/back/api_endpoints.md` | MINOR |
| DB schema change | `features/<name>/back/data_model.md` | MINOR |
| UI change | `features/<name>/front/pages.md` + `ui/wireframes.md` | PATCH |
| Workflow change | `features/<name>/workflows/*.md` | PATCH |
| Bug fix | `context_packs/memory/known_issues.md` | PATCH |
| Architecture decision | `context_packs/memory/decisions_log.md` | PATCH |
| Convention change | This file (`conventions.md`) | PATCH |

### Git Push After Every Change
```bash
cd ai_factory_prompt_repo
git add -A
git commit -m "<type>(<scope>): <description>"
git push origin main
```

---

## Code Style

### TypeScript (Frontend + Backend Services)
- **Formatter:** Prettier (2-space indent, single quotes, trailing commas)
- **Linter:** ESLint with strict TypeScript rules
- **Imports:** Absolute paths using `@/` alias
- **Types:** Prefer `interface` over `type` for object shapes; use `type` for unions/intersections
- **Null handling:** Use `strictNullChecks`, prefer `undefined` over `null`
- **Async:** Always use async/await, never raw Promises with `.then()`

### Python (Orchestrator + Planning Service)
- **Formatter:** Black (88 line length)
- **Linter:** Ruff
- **Type hints:** Required for all function signatures
- **Imports:** isort with Black-compatible settings
- **Async:** Use `asyncio` with `async/await`

## Naming Conventions

| Entity | Convention | Example |
|--------|-----------|---------|
| Files (component) | PascalCase | `ExecutionDAG.tsx` |
| Files (module) | kebab-case | `execution-service.ts` |
| Files (Python) | snake_case | `execution_service.py` |
| Variables | camelCase | `executionId` |
| Constants | UPPER_SNAKE | `MAX_RETRY_COUNT` |
| Types/Interfaces | PascalCase | `ExecutionStep` |
| Database tables | snake_case (plural) | `executions` |
| Database columns | snake_case | `created_at` |
| API endpoints | kebab-case | `/api/v1/execution-steps` |
| Environment vars | UPPER_SNAKE | `DATABASE_URL` |
| Git branches | kebab-case | `feature/execution-dag` |
| Commits | Conventional | `feat(orchestrator): add retry logic` |

## Git Workflow

### Branch Strategy
```
main          ← Production-ready, protected
  └── develop ← Integration branch
       ├── feature/xxx  ← New features
       ├── fix/xxx      ← Bug fixes
       └── refactor/xxx ← Code improvements
```

### Commit Messages (Conventional Commits)
```
<type>(<scope>): <description>

[optional body]

[optional footer(s)]
```

Types: `feat`, `fix`, `refactor`, `docs`, `test`, `chore`, `perf`, `ci`
Scopes: `orchestrator`, `planning`, `command-ui`, `agents`, `knowledge`, `feedback`, `shared`

### PR Rules
- Every PR requires at least 1 review
- All tests must pass
- No decrease in coverage
- Squash merge to develop, merge commit to main

## API Conventions
- RESTful with JSON bodies
- Versioned: `/api/v1/...`
- Pagination: `?page=1&limit=20` → `{ data: [], meta: { total, page, limit } }`
- Errors: `{ error: { code, message, details?, requestId } }`
- Dates: ISO 8601 UTC (`2025-01-15T10:30:00Z`)
- IDs: UUIDv4

## File Organization Rules
1. One component per file (React)
2. Co-locate tests with source (`*.test.ts` next to `*.ts`)
3. Shared types in `packages/shared/types/`
4. Feature-specific types in feature directory
5. No circular imports (enforced by ESLint)
6. Max file length: 300 lines (split if larger)

## Documentation Rules
1. Every public function has JSDoc/docstring
2. Every module has a README.md
3. Every API endpoint documented in OpenAPI
4. Every decision in ADR format
5. Every known issue tracked in `known_issues.md`
