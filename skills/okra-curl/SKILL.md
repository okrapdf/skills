---
name: okra-curl
description: HTTP/curl cookbook for the OkraPDF REST API — upload PDFs, chat completions, structured extraction, collections, and exports.
---

# OkraPDF HTTP API

Direct HTTP access to OkraPDF. OpenAI-compatible chat completions, REST endpoints for everything.

## Auth

All authenticated endpoints require:
```
Authorization: Bearer YOUR_API_KEY
```

Base URL: `https://api.okrapdf.com`

## Upload a PDF

### From URL
```bash
curl -X POST https://api.okrapdf.com/v1/documents \
  -H "Authorization: Bearer $OKRA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://arxiv.org/pdf/2307.09288"}'
```

### From file (multipart)
```bash
curl -X POST https://api.okrapdf.com/v1/documents \
  -H "Authorization: Bearer $OKRA_API_KEY" \
  -F "file=@report.pdf" \
  -F "page_images=cover"
```

Upload options (form fields or JSON):
- `page_images` — `none`, `cover` (default), `lazy`
- `processor` — `textlayer`, `llamaparse`, `unstructured`, `azure-di`
- `strategy` — processing strategy override

Response:
```json
{
  "document_id": "doc-abc123",
  "phase": "extracting",
  "pages_total": 42
}
```

## Check Status

```bash
curl https://api.okrapdf.com/v1/documents/doc-abc123/status \
  -H "Authorization: Bearer $OKRA_API_KEY"
```

Response: `{"phase": "complete", "page_count": 42, "total_nodes": 318}`

## Read Content

### Full document (markdown)
```bash
curl https://api.okrapdf.com/v1/documents/doc-abc123/full.md \
  -H "Authorization: Bearer $OKRA_API_KEY"
```

### Specific pages
```bash
curl "https://api.okrapdf.com/v1/documents/doc-abc123/pages/3" \
  -H "Authorization: Bearer $OKRA_API_KEY"
```

### All pages as JSON
```bash
curl https://api.okrapdf.com/v1/documents/doc-abc123/pages \
  -H "Authorization: Bearer $OKRA_API_KEY"
```

## Chat Completions (OpenAI-compatible)

### Ask a question about a document
```bash
curl -X POST https://api.okrapdf.com/document/doc-abc123/chat/completions \
  -H "Authorization: Bearer $OKRA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [{"role": "user", "content": "What was total revenue in 2024?"}],
    "stream": false
  }'
```

### Structured extraction (JSON schema)
```bash
curl -X POST https://api.okrapdf.com/document/doc-abc123/chat/completions \
  -H "Authorization: Bearer $OKRA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [{"role": "user", "content": "Extract all line items from this invoice"}],
    "response_format": {
      "type": "json_schema",
      "json_schema": {
        "name": "invoice",
        "schema": {
          "type": "object",
          "properties": {
            "line_items": {
              "type": "array",
              "items": {
                "type": "object",
                "properties": {
                  "description": {"type": "string"},
                  "amount": {"type": "number"}
                }
              }
            },
            "total": {"type": "number"}
          }
        },
        "strict": true
      }
    },
    "stream": false
  }'
```

### Streaming
```bash
curl -X POST https://api.okrapdf.com/document/doc-abc123/chat/completions \
  -H "Authorization: Bearer $OKRA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [{"role": "user", "content": "Summarize this document"}],
    "stream": true
  }'
```

## Exports

```bash
# Markdown
curl https://api.okrapdf.com/exports/doc-abc123/markdown \
  -H "Authorization: Bearer $OKRA_API_KEY"

# Excel
curl -o report.xlsx https://api.okrapdf.com/exports/doc-abc123/excel \
  -H "Authorization: Bearer $OKRA_API_KEY"

# DOCX
curl -o report.docx https://api.okrapdf.com/exports/doc-abc123/docx \
  -H "Authorization: Bearer $OKRA_API_KEY"

# JSON snapshot
curl https://api.okrapdf.com/exports/doc-abc123/snapshot \
  -H "Authorization: Bearer $OKRA_API_KEY"
```

## Page Images

Deterministic URLs, CDN-cached:

```bash
# Page 1 image
curl -o page1.png https://res.okrapdf.com/v1/documents/doc-abc123/pg_1.png

# With resize transform
curl -o thumb.png "https://res.okrapdf.com/v1/documents/doc-abc123/w_400,h_300/pg_1.png"
```

## Tables and Entities

```bash
# List tables
curl https://api.okrapdf.com/v1/documents/doc-abc123/entities/tables \
  -H "Authorization: Bearer $OKRA_API_KEY"

# List all entities
curl https://api.okrapdf.com/v1/documents/doc-abc123/entities \
  -H "Authorization: Bearer $OKRA_API_KEY"
```

## Collections

Group documents and query across all of them at once.

### Create (with optional seed documents)
```bash
curl -X POST https://api.okrapdf.com/v1/collections \
  -H "Authorization: Bearer $OKRA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Q4 Earnings",
    "description": "Quarterly earnings reports",
    "document_ids": ["doc-abc123", "doc-def456"]
  }'
```

### Add documents
```bash
curl -X POST https://api.okrapdf.com/v1/collections/col-xxx/documents \
  -H "Authorization: Bearer $OKRA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"document_ids": ["doc-ghi789"]}'
```

### Query across documents

Two query modes:

| Mode | Behavior | Best for |
|------|----------|----------|
| `fanout` (default) | Separate completion per document, NDJSON stream | Per-document answers, structured extraction |
| `sandbox` | Single LLM with grep/Python over all docs | Cross-document search, comparisons, aggregation |

**Fanout (default):**
```bash
curl -X POST https://api.okrapdf.com/v1/collections/col-xxx/query \
  -H "Authorization: Bearer $OKRA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "What was total revenue in Q4?"}'
```

Fanout NDJSON response events:
```
{"type":"start","query_id":"...","doc_count":7}
{"type":"result","doc_id":"doc-xxx","answer":"Apple reported revenue of..."}
{"type":"done","completed":7,"failed":0}
```

**Sandbox mode:**
```bash
curl -X POST https://api.okrapdf.com/v1/collections/col-xxx/query \
  -H "Authorization: Bearer $OKRA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "Compare R&D spending across all companies. Show a table.",
    "mode": "sandbox"
  }'
```

Sandbox returns a single JSON response:
```json
{
  "answer": "Based on the 10-K filings...",
  "model": "moonshotai/kimi-k2.5",
  "mode": "sandbox",
  "usage": {"inputTokens": 19364, "outputTokens": 1804, "costUsd": 0.005},
  "tool_calls": 5,
  "duration_ms": 78166
}
```

### Get collection
```bash
curl https://api.okrapdf.com/v1/collections/col-xxx \
  -H "Authorization: Bearer $OKRA_API_KEY"
```

### List collections
```bash
curl https://api.okrapdf.com/v1/collections \
  -H "Authorization: Bearer $OKRA_API_KEY"
```

### Remove documents
```bash
curl -X DELETE https://api.okrapdf.com/v1/collections/col-xxx/documents \
  -H "Authorization: Bearer $OKRA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"document_ids": ["doc-abc123"]}'
```

### Export collection
```bash
# NDJSON stream (default)
curl -N "https://api.okrapdf.com/v1/collections/col-xxx/export?format=markdown" \
  -H "Authorization: Bearer $OKRA_API_KEY"

# Zip archive
curl -L "https://api.okrapdf.com/v1/collections/col-xxx/export?format=zip" \
  -H "Authorization: Bearer $OKRA_API_KEY" -o collection.zip
```

### Delete collection (preserves documents)
```bash
curl -X DELETE https://api.okrapdf.com/v1/collections/col-xxx \
  -H "Authorization: Bearer $OKRA_API_KEY"
```

### Fan-out pattern (scripting)

For maximum control, query individual documents in parallel:
```bash
for doc_id in doc-abc123 doc-def456 doc-ghi789; do
  curl -s -X POST "https://api.okrapdf.com/document/$doc_id/chat/completions" \
    -H "Authorization: Bearer $OKRA_API_KEY" \
    -H "Content-Type: application/json" \
    -d "{\"messages\": [{\"role\": \"user\", \"content\": \"What was total revenue?\"}], \"stream\": false}" &
done
wait
```

## List Documents

```bash
curl "https://api.okrapdf.com/v1/documents?limit=20" \
  -H "Authorization: Bearer $OKRA_API_KEY"
```

Pagination: `?limit=20&after=cursor_token`

## Available Vendors

```bash
curl https://api.okrapdf.com/v1/vendors
```

Returns list of available OCR processors with capabilities and pricing tier.

## Error Handling

| Status | Meaning |
|--------|---------|
| 202 | Accepted, processing async |
| 400 | Bad request (check body) |
| 401 | Missing or invalid API key |
| 404 | Document not found |
| 409 | Conflict (document already exists) |
| 429 | Rate limited |
| 500 | Server error |
