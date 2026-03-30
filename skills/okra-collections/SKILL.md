---
name: okra-collections
description: Multi-document queries with OkraPDF collections — group PDFs, query across them, get cross-document analysis with citations.
---

# OkraPDF Collections

Group documents into collections and query across all of them at once. Fan-out Q&A with cross-document synthesis.

## Concept

A collection is a named group of OkraPDF documents. When you query a collection, OkraPDF fans out the question to every document in parallel, then synthesizes a unified answer with per-document citations.

## Create a Collection

### CLI
```bash
okra collections create --name "SEC 10-K Filings 2024"
```

### HTTP
```bash
curl -X POST https://api.okrapdf.com/v1/collections \
  -H "Authorization: Bearer $OKRA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "SEC 10-K Filings 2024"}'
```

### MCP
Collections are managed via CLI or HTTP. MCP tools operate on individual documents.

## Add Documents

```bash
# CLI
okra collections add-docs col-xxx doc-abc123 doc-def456 doc-ghi789

# HTTP
curl -X POST https://api.okrapdf.com/v1/collections/col-xxx/documents \
  -H "Authorization: Bearer $OKRA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"document_ids": ["doc-abc123", "doc-def456"]}'
```

## Query Across Documents

### CLI
```bash
okra collections query col-xxx "Compare revenue growth across all companies"
```

### HTTP (NDJSON streaming)
```bash
curl -X POST https://api.okrapdf.com/v1/collections/col-xxx/query \
  -H "Authorization: Bearer $OKRA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"prompt": "What was each company total revenue in FY2024?"}'
```

Response streams NDJSON events:
```
{"event": "start", "documents": 7}
{"event": "text_delta", "document_id": "doc-abc123", "delta": "Apple reported..."}
{"event": "result", "document_id": "doc-abc123", "answer": "..."}
{"event": "result", "document_id": "doc-def456", "answer": "..."}
{"event": "done", "synthesis": "Across all 7 companies..."}
```

## Fan-out Pattern (Scripting)

For maximum control, query individual documents and aggregate:

```bash
# Query same question across multiple docs
for doc_id in doc-abc123 doc-def456 doc-ghi789; do
  curl -s -X POST "https://api.okrapdf.com/document/$doc_id/chat/completions" \
    -H "Authorization: Bearer $OKRA_API_KEY" \
    -H "Content-Type: application/json" \
    -d "{\"messages\": [{\"role\": \"user\", \"content\": \"What was total revenue?\"}], \"stream\": false}" &
done
wait
```

## Public Collections

Some collections are public (no auth needed to read):

| Collection | ID | Description |
|---|---|---|
| FinanceBench | `col-1f2b793609f042e882ba1c0f8598f902` | 10 SEC 10-Ks for financial Q&A benchmark |
| Mag7 | `col-40da068481cf4f248853507cba6be611` | Magnificent 7 tech company 10-Ks |

```bash
# Query public collection
curl https://api.okrapdf.com/v1/collections/col-1f2b793609f042e882ba1c0f8598f902 \
  -H "Authorization: Bearer $OKRA_API_KEY"
```

## Manage Collections

```bash
# List all collections
okra collections list

# Show collection details (documents, metadata)
okra collections show col-xxx

# Remove documents
okra collections remove-docs col-xxx doc-abc123

# Delete collection
okra collections delete col-xxx

# Export all collection data
okra collections export col-xxx
```

## Use Cases

- **Financial analysis**: Upload 10-Ks from multiple companies, compare metrics
- **Legal review**: Upload contracts, find conflicting clauses across documents
- **Research synthesis**: Upload papers, ask cross-cutting questions
- **Compliance audit**: Upload policies + filings, check for gaps
