# Knowledge Base — API Endpoints

## Base Path: `/api/v1/knowledge`

### Indexing

#### POST /api/v1/knowledge/index
Trigger indexing of a repository or codebase.

**Request:**
```json
{
  "projectId": "proj-uuid",
  "source": {
    "type": "git_repo",
    "url": "https://github.com/org/repo.git",
    "branch": "main",
    "paths": ["src/", "docs/"]
  },
  "options": {
    "chunkSize": 512,
    "overlap": 50,
    "includeMetadata": true,
    "languages": ["typescript", "python", "markdown"]
  }
}
```

**Response:**
```json
{
  "jobId": "idx-uuid",
  "status": "processing",
  "estimatedFiles": 245,
  "estimatedChunks": 1200
}
```

#### GET /api/v1/knowledge/index/{jobId}
Check indexing job status.

#### DELETE /api/v1/knowledge/index/{projectId}
Remove all indexed data for a project.

### Search

#### POST /api/v1/knowledge/search
Semantic search across indexed content.

**Request:**
```json
{
  "query": "How does the authentication middleware validate JWT tokens?",
  "projectId": "proj-uuid",
  "filters": {
    "fileTypes": ["typescript"],
    "paths": ["src/middleware/"],
    "minScore": 0.7
  },
  "limit": 10,
  "includeContext": true
}
```

**Response:**
```json
{
  "results": [
    {
      "score": 0.92,
      "file": "src/middleware/auth.ts",
      "chunk": "export async function validateToken(token: string)...",
      "lineStart": 45,
      "lineEnd": 78,
      "context": { "before": "...", "after": "..." },
      "metadata": { "lastModified": "2025-02-15", "author": "dev-1" }
    }
  ],
  "totalMatches": 8,
  "searchTime": 120
}
```

#### POST /api/v1/knowledge/search/hybrid
Combined semantic + keyword search for higher precision.

### Context Assembly

#### POST /api/v1/knowledge/context
Assemble optimal context for an agent task (RAG).

**Request:**
```json
{
  "task": "Implement password reset endpoint",
  "projectId": "proj-uuid",
  "maxTokens": 50000,
  "requiredFiles": ["shared/backend.md"],
  "strategy": "relevance_first"
}
```

**Response:**
```json
{
  "context": [
    { "source": "required", "file": "shared/backend.md", "tokens": 3200 },
    { "source": "semantic", "file": "src/auth/login.ts", "tokens": 1500, "relevance": 0.91 },
    { "source": "semantic", "file": "src/email/templates.ts", "tokens": 800, "relevance": 0.85 }
  ],
  "totalTokens": 12400,
  "strategy": "relevance_first"
}
```
