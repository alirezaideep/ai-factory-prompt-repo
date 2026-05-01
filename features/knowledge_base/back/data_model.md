# Knowledge Base — Data Model

## Note
Primary tables and vector store schemas are defined in feature_brief.md. This file covers indexing pipeline details and search optimization.

## Indexing Pipeline

### Step 1: Document Ingestion
```python
async def ingest_document(project_id: str, path: str, content: str, metadata: dict):
    """
    1. Compute checksum
    2. Check if already indexed (skip if unchanged)
    3. Chunk document
    4. Generate embeddings
    5. Upsert to vector store
    6. Update indexed_documents table
    """
    checksum = sha256(content)
    
    existing = await db.get_indexed_document(project_id, path)
    if existing and existing.checksum == checksum:
        return  # No changes, skip
    
    # Remove old chunks
    if existing:
        await vector_store.delete(filter={"path": path, "project_id": project_id})
    
    # Chunk
    chunks = chunk_document(content, path, metadata)
    
    # Embed
    embeddings = await embedding_model.embed_batch([c.text for c in chunks])
    
    # Upsert
    points = [
        {
            "id": generate_chunk_id(project_id, path, i),
            "vector": embeddings[i],
            "payload": {
                "project_id": project_id,
                "path": path,
                "section": chunks[i].section,
                "module": metadata.get("module"),
                "type": metadata.get("type"),
                "version": metadata.get("version"),
                "chunk_index": i,
                "token_count": chunks[i].token_count,
                "text": chunks[i].text
            }
        }
        for i in range(len(chunks))
    ]
    await vector_store.upsert("repo_documents", points)
    
    # Update tracking table
    await db.upsert_indexed_document(project_id, path, checksum, len(chunks))
```

### Step 2: Chunking Strategy
```python
def chunk_document(content: str, path: str, metadata: dict) -> list[Chunk]:
    """
    Strategy depends on file type:
    - .md files: Split by ## headers
    - .ts/.py files: Split by function/class
    - .sql files: Split by CREATE/ALTER statements
    - .yaml/.json: Split by top-level keys
    """
    file_ext = path.split(".")[-1]
    
    if file_ext == "md":
        return chunk_by_headers(content, max_tokens=1000, overlap=100)
    elif file_ext in ("ts", "py", "js"):
        return chunk_by_code_blocks(content, max_tokens=800, overlap=50)
    elif file_ext == "sql":
        return chunk_by_statements(content, max_tokens=500)
    else:
        return chunk_by_tokens(content, max_tokens=1000, overlap=100)
```

### Step 3: Search with Re-ranking
```python
async def search(query: str, filters: dict, max_results: int, max_tokens: int) -> list:
    """
    Hybrid search: semantic (70%) + keyword (30%)
    Then re-rank with cross-encoder for precision.
    Then trim to token budget.
    """
    # Semantic search
    query_embedding = await embedding_model.embed(query)
    semantic_results = await vector_store.search(
        collection="repo_documents",
        vector=query_embedding,
        filter=filters,
        limit=max_results * 3  # Over-fetch for re-ranking
    )
    
    # Keyword search (BM25)
    keyword_results = await bm25_search(query, filters, limit=max_results * 2)
    
    # Merge and deduplicate
    merged = merge_results(semantic_results, keyword_results, alpha=0.7)
    
    # Re-rank
    reranked = await cross_encoder.rerank(query, merged, top_k=max_results * 2)
    
    # Trim to token budget
    final = []
    tokens_used = 0
    for result in reranked:
        if tokens_used + result.token_count > max_tokens:
            break
        final.append(result)
        tokens_used += result.token_count
    
    return final[:max_results]
```

## Incremental Indexing Trigger

```python
# Called by Prompt Factory after any repo update
async def on_repo_updated(project_id: str, changed_files: list[str]):
    """Incrementally re-index only changed files."""
    for path in changed_files:
        content = await repo.read_file(project_id, path)
        metadata = infer_metadata(path)
        await ingest_document(project_id, path, content, metadata)
    
    logger.info(f"Re-indexed {len(changed_files)} files for project {project_id}")
```

## Pattern Storage

```python
async def store_successful_pattern(execution_id: str, step_id: str):
    """
    After successful execution, extract and store reusable patterns.
    """
    output = await db.get_step_output(execution_id, step_id)
    
    if not output or not output.get("success"):
        return
    
    # Extract code patterns from generated files
    for file in output.get("files", []):
        if file["operation"] == "created" and is_code_file(file["path"]):
            pattern = {
                "name": generate_pattern_name(file["path"], output["summary"]),
                "language": detect_language(file["path"]),
                "framework": detect_framework(file["content"]),
                "code": file["content"],
                "description": output["summary"],
                "tags": extract_tags(file["content"], output["summary"]),
                "source_execution_id": execution_id
            }
            
            # Embed and store
            embedding = await embedding_model.embed(pattern["description"] + "\n" + pattern["code"][:500])
            await vector_store.upsert("code_patterns", [{
                "id": generate_id(),
                "vector": embedding,
                "payload": pattern
            }])
            
            await db.insert_code_pattern(pattern)
```
