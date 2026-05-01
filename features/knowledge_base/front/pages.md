# Knowledge Base — Frontend Pages

## Page: Knowledge Explorer (`/knowledge`)

### Layout
Search-centric page with results panel and file preview.

### Components
- **SearchBar** — Natural language query input with filters dropdown
- **ResultsList** — Ranked results with relevance score, file path, code preview
- **FilePreview** — Side panel showing full file with highlighted match
- **IndexStatus** — Shows indexing progress and last update time per project

### Data Flow
```
trpc.knowledge.search.useMutation()
trpc.knowledge.indexStatus.useQuery({ projectId })
trpc.knowledge.reindex.useMutation()
```

---

## Page: Index Management (`/knowledge/indexes`)

### Layout
Table of all indexed projects with status, size, and actions.

### Columns
- Project name
- Files indexed
- Chunks count
- Last indexed
- Status (up-to-date, stale, indexing, error)
- Actions (reindex, delete, configure)

### Features
- Trigger manual reindex
- Configure indexing options per project
- View indexing logs
- Monitor storage usage
