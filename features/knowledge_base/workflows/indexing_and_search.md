# Knowledge Base — Indexing & Search Workflow

## Overview

The Knowledge Base provides contextual intelligence to all agents by indexing prompt repositories, code patterns, and execution history. It uses hybrid search (semantic + keyword) with re-ranking.

## Indexing Triggers

| Trigger | Action |
|---------|--------|
| Repo commit (version bump) | Re-index changed files |
| Successful execution | Extract and store code patterns |
| Human correction | Store correction as negative example |
| New project created | Full index of initial skeleton |
| Manual re-index request | Full re-index of entire repo |

## Search Flow

```
[Agent needs context]
    ↓
[Context Assembler calls Knowledge Base]
    query: task description
    filters: {project_id, module, layer, file_type}
    max_tokens: remaining budget after loading formula
    ↓
[Hybrid Search]
    ├── Semantic: embed query → cosine similarity (weight: 0.7)
    └── Keyword: BM25 full-text search (weight: 0.3)
    ↓
[Merge & Deduplicate results]
    ↓
[Cross-Encoder Re-ranking]
    top_k = requested * 2 → re-rank → take top_k
    ↓
[Token Budget Trimming]
    Greedily add results until budget exhausted
    ↓
[Return ranked chunks with metadata]
```

## Collections (Vector Store)

### Collection: `repo_documents`
- **Content:** Chunks from prompt repository files
- **Use case:** Find relevant documentation for a task
- **Embedding model:** text-embedding-3-small (1536 dims)
- **Metadata:** project_id, path, module, layer, file_type, version, section

### Collection: `code_patterns`
- **Content:** Successful code snippets from past executions
- **Use case:** Find similar implementations for reference
- **Embedding model:** text-embedding-3-small (1536 dims)
- **Metadata:** language, framework, pattern_type, success_rate, source_project

### Collection: `execution_history`
- **Content:** Task descriptions + outcomes from past executions
- **Use case:** Estimate difficulty, find similar past tasks
- **Embedding model:** text-embedding-3-small (1536 dims)
- **Metadata:** agent_type, success, complexity, tokens_used, duration

## Chunking Strategies

### Markdown Files (repo documents)
```python
def chunk_markdown(content: str) -> list[Chunk]:
    """
    Split by ## headers. Each chunk = one section.
    If section > 1000 tokens, split by ### sub-headers.
    If still > 1000 tokens, split by paragraphs with overlap.
    """
    sections = split_by_headers(content, level=2)
    chunks = []
    
    for section in sections:
        if count_tokens(section.text) <= 1000:
            chunks.append(Chunk(
                text=section.text,
                section=section.heading,
                token_count=count_tokens(section.text)
            ))
        else:
            # Split further
            sub_sections = split_by_headers(section.text, level=3)
            for sub in sub_sections:
                if count_tokens(sub.text) <= 1000:
                    chunks.append(Chunk(text=sub.text, section=sub.heading))
                else:
                    # Last resort: paragraph split with overlap
                    para_chunks = split_by_paragraphs(sub.text, max_tokens=800, overlap=100)
                    chunks.extend(para_chunks)
    
    return chunks
```

### Code Files
```python
def chunk_code(content: str, language: str) -> list[Chunk]:
    """
    Split by function/class boundaries.
    Each chunk = one function or class (with imports context).
    """
    tree = parse_ast(content, language)
    imports = extract_imports(tree)
    
    chunks = []
    for node in tree.top_level_nodes:
        if node.type in ('function', 'class', 'interface'):
            chunk_text = f"{imports}\n\n{node.text}"
            chunks.append(Chunk(
                text=chunk_text,
                section=f"{node.type}:{node.name}",
                token_count=count_tokens(chunk_text)
            ))
    
    return chunks
```

## Pattern Learning Pipeline

```
[Execution completes successfully]
    ↓
[Extract patterns]
    ├── Code patterns (functions, components, routes)
    ├── Architecture patterns (file structure, imports)
    └── Prompt patterns (what worked well)
    ↓
[Score pattern quality]
    ├── Tests passed? (+2)
    ├── Review approved without changes? (+3)
    ├── Human correction needed? (-2)
    └── Reused in future tasks? (+1 per reuse)
    ↓
[Store in code_patterns collection]
    ↓
[Periodic cleanup]
    ├── Remove patterns with score < -3
    ├── Merge similar patterns (cosine > 0.95)
    └── Promote high-score patterns to "golden examples"
```

## Cache Strategy

```python
class SearchCache:
    """
    Cache search results to avoid redundant embedding calls.
    Invalidation: on repo version bump.
    """
    
    TTL = 3600  # 1 hour
    
    def cache_key(self, query: str, filters: dict) -> str:
        return hashlib.md5(f"{query}:{json.dumps(filters, sort_keys=True)}".encode()).hexdigest()
    
    async def get_or_search(self, query: str, filters: dict, **kwargs):
        key = self.cache_key(query, filters)
        
        # Check cache
        cached = await redis.get(f"kb:search:{key}")
        if cached:
            return json.loads(cached)
        
        # Perform search
        results = await self._search(query, filters, **kwargs)
        
        # Cache results
        await redis.setex(f"kb:search:{key}", self.TTL, json.dumps(results))
        
        return results
    
    async def invalidate_project(self, project_id: str):
        """Invalidate all cached results for a project."""
        pattern = f"kb:search:*{project_id}*"
        keys = await redis.keys(pattern)
        if keys:
            await redis.delete(*keys)
```
