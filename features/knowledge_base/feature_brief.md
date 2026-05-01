# Knowledge Base — Feature Brief

## Module Identity

| Field | Value |
|-------|-------|
| Module ID | `knowledge_base` |
| Layer | 7 (Intelligence) |
| Owner | Backend Team Lead |
| Status | Planned |
| Priority | High (learning system) |

## Purpose

The Knowledge Base provides **organizational memory** for the AI Software Factory. It stores embeddings of all repository files, past execution results, code patterns, and decisions. It enables RAG (Retrieval-Augmented Generation) for agents, similarity search for planning, and historical analysis for the feedback engine.

## Core Capabilities

### 1. Document Indexing
- Index all Prompt Repository files (Markdown → embeddings)
- Index generated code files (for pattern matching)
- Index execution logs (for learning from past runs)
- Incremental re-indexing on repository updates
- Multi-modal: text, code, diagrams (via description)

### 2. RAG (Retrieval-Augmented Generation)
- Semantic search across all indexed documents
- Hybrid search: semantic + keyword (BM25)
- Context-aware retrieval (filter by module, file type, date range)
- Relevance scoring with confidence levels
- Token-budget-aware retrieval (return top-K within budget)

### 3. Code Pattern Library
- Store successful code patterns from past executions
- Categorize by: language, framework, pattern type, module
- Similarity matching for new tasks
- Pattern evolution tracking (how patterns improve over time)
- Anti-pattern detection (patterns that led to failures)

### 4. Decision Memory
- Store all planning decisions and their outcomes
- Store approval decisions with rationale
- Store human feedback and corrections
- Enable "why was this done?" queries
- Support decision reversal tracking

### 5. Historical Analytics
- Success rate by agent type, module, task complexity
- Average token usage by task type
- Common failure patterns
- Time-to-completion trends
- Cost optimization opportunities

## Architecture

### Embedding Pipeline
```
[Source Document] → [Chunking] → [Embedding Model] → [Vector Store]
                                                         ↓
                                                    [Metadata Store]

Chunking Strategy:
- Markdown files: Split by ## headers (preserve section integrity)
- Code files: Split by function/class boundaries
- Logs: Split by execution step
- Max chunk size: 1000 tokens
- Overlap: 100 tokens between chunks
```

### Search Pipeline
```
[Query] → [Embed Query] → [Vector Search (top-50)]
                                    ↓
                          [Re-rank (cross-encoder)]
                                    ↓
                          [Filter by metadata]
                                    ↓
                          [Token budget trim]
                                    ↓
                          [Return top-K results]
```

## API Endpoints

### POST /api/v1/knowledge/index
Index a document or batch of documents.
```json
// Request
{
  "documents": [
    {
      "id": "doc_123",
      "path": "features/orders/back/api_endpoints.md",
      "content": "...",
      "metadata": {
        "module": "order_management",
        "type": "api_spec",
        "version": "v1.3.0",
        "project_id": "proj_abc"
      }
    }
  ]
}

// Response 200
{
  "indexed": 1,
  "chunks_created": 8,
  "total_tokens": 3200
}
```

### POST /api/v1/knowledge/search
Semantic search across knowledge base.
```json
// Request
{
  "query": "How to add a new field to the orders table with migration",
  "filters": {
    "module": "order_management",
    "type": ["api_spec", "data_model", "code_pattern"]
  },
  "max_results": 10,
  "max_tokens": 5000,
  "include_metadata": true
}

// Response 200
{
  "results": [
    {
      "id": "chunk_456",
      "path": "features/orders/back/data_model.md",
      "section": "## Migration Strategy",
      "content": "...",
      "score": 0.92,
      "metadata": {"module": "order_management", "version": "v1.3.0"},
      "tokens": 450
    }
  ],
  "total_tokens": 3200,
  "search_time_ms": 45
}
```

### POST /api/v1/knowledge/find-similar
Find similar past tasks/executions.
```json
// Request
{
  "task_description": "Add urgent order priority with notification",
  "top_k": 5
}

// Response 200
{
  "similar_tasks": [
    {
      "task_id": "task_old_123",
      "title": "Add VIP customer priority",
      "similarity_score": 0.87,
      "outcome": "success",
      "execution_time_minutes": 18,
      "tokens_used": 15000,
      "plan_steps": 8
    }
  ]
}
```

### POST /api/v1/knowledge/store-pattern
Store a successful code pattern.
```json
{
  "pattern_name": "add_column_with_migration",
  "language": "typescript",
  "framework": "drizzle",
  "code": "...",
  "description": "Pattern for adding a new column with proper migration",
  "tags": ["database", "migration", "schema"],
  "source_execution_id": "exec_abc",
  "success_rate": 0.95
}
```

### GET /api/v1/knowledge/analytics
Get historical analytics.

### POST /api/v1/knowledge/reindex
Trigger full or partial reindex.

## Data Model

### Vector Store: Qdrant
```
Collection: repo_documents
  - vector: 1536 dimensions (OpenAI ada-002) or 768 (local model)
  - payload:
    - path: string
    - section: string
    - module: string
    - type: string (api_spec | data_model | workflow | code | test | docs)
    - version: string
    - project_id: string
    - chunk_index: integer
    - token_count: integer
    - created_at: datetime

Collection: code_patterns
  - vector: 1536 dimensions
  - payload:
    - pattern_name: string
    - language: string
    - framework: string
    - tags: string[]
    - success_rate: float
    - usage_count: integer
    - source_execution_id: string

Collection: execution_history
  - vector: 1536 dimensions
  - payload:
    - task_description: string
    - outcome: string (success | partial | failed)
    - agent_type: string
    - tokens_used: integer
    - duration_ms: integer
    - error_type: string (if failed)
```

### PostgreSQL Tables

```sql
CREATE TABLE indexed_documents (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID NOT NULL REFERENCES projects(id),
  path VARCHAR(500) NOT NULL,
  version VARCHAR(20) NOT NULL,
  
  chunk_count INTEGER NOT NULL,
  total_tokens INTEGER NOT NULL,
  
  last_indexed_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  checksum VARCHAR(64) NOT NULL,
  
  UNIQUE(project_id, path, version)
);

CREATE TABLE code_patterns (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(100) NOT NULL,
  language VARCHAR(30) NOT NULL,
  framework VARCHAR(50),
  
  code TEXT NOT NULL,
  description TEXT NOT NULL,
  tags TEXT[] DEFAULT '{}',
  
  success_rate DECIMAL(3,2) NOT NULL DEFAULT 1.0,
  usage_count INTEGER NOT NULL DEFAULT 0,
  last_used_at TIMESTAMP WITH TIME ZONE,
  
  source_execution_id UUID REFERENCES executions(id),
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_code_patterns_tags ON code_patterns USING GIN(tags);
CREATE INDEX idx_code_patterns_language ON code_patterns(language);
```

## Embedding Strategy

| Content Type | Model | Dimensions | Chunking |
|--------------|-------|------------|----------|
| Markdown docs | OpenAI text-embedding-3-small | 1536 | By ## headers |
| Code files | CodeBERT or StarCoder embeddings | 768 | By function |
| Execution logs | OpenAI text-embedding-3-small | 1536 | By step |
| Decisions | OpenAI text-embedding-3-small | 1536 | Whole document |

## Configuration

```yaml
knowledge_base:
  vector_store:
    provider: "qdrant"
    url: "http://qdrant:6333"
    collections:
      repo_documents:
        vector_size: 1536
        distance: "cosine"
      code_patterns:
        vector_size: 1536
        distance: "cosine"
      execution_history:
        vector_size: 1536
        distance: "cosine"
  
  embedding:
    model: "text-embedding-3-small"
    batch_size: 100
    max_retries: 3
  
  search:
    default_top_k: 10
    rerank_enabled: true
    rerank_model: "cross-encoder/ms-marco-MiniLM-L-6-v2"
    hybrid_alpha: 0.7  # 0.7 semantic + 0.3 keyword
  
  indexing:
    auto_reindex_on_commit: true
    full_reindex_schedule: "0 2 * * 0"  # Weekly Sunday 2 AM
    max_chunk_tokens: 1000
    chunk_overlap_tokens: 100

  retention:
    execution_history_days: 365
    pattern_min_success_rate: 0.5  # Remove patterns below this
```

## Success Metrics

| Metric | Target |
|--------|--------|
| Search relevance (MRR@10) | > 0.8 |
| Indexing latency | < 5 seconds per document |
| Search latency (p95) | < 200ms |
| Pattern reuse rate | > 30% of similar tasks |
| Storage efficiency | < 10GB per project |
