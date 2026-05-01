# Project Conventions

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
