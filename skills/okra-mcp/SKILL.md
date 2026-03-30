---
name: okra-mcp
description: Connect to OkraPDF via MCP to upload PDFs, read extracted content, ask questions, and extract structured data with JSON schemas.
---

# OkraPDF MCP

Upload a PDF, get an API. Extract tables, ask questions, get structured JSON — all through MCP.

## Setup

Add to your agent's MCP config:

```json
{
  "mcpServers": {
    "okra-pdf": {
      "type": "url",
      "url": "https://api.okrapdf.com/mcp",
      "headers": { "Authorization": "Bearer YOUR_API_KEY" }
    }
  }
}
```

Get an API key at [okrapdf.com](https://okrapdf.com). Free tier includes $5 credit.

## Available Tools

### upload_document
Upload a PDF from a URL. Returns document ID and processing status.

```
upload_document(url: "https://example.com/report.pdf")
upload_document(url: "https://arxiv.org/pdf/2307.09288", wait: true)
upload_document(url: "https://example.com/invoice.pdf", page_images: "lazy")
```

Parameters:
- `url` (required) — Public URL of the PDF
- `wait` (default: true) — Block until extraction completes
- `document_id` (optional) — Custom ID (auto-generated if omitted)
- `page_images` — `none`, `cover` (default), or `lazy` (on-demand rendering)
- `processor` — `textlayer`, `llamaparse`, `unstructured`, `azure-di`, `parse-proxy`

### read_document
Get extracted markdown content. Supports page ranges for large documents.

```
read_document(document_id: "doc-abc123")
read_document(document_id: "doc-abc123", pages: "1-5")
read_document(document_id: "https://arxiv.org/pdf/2307.09288")
```

Parameters:
- `document_id` (required) — Document ID, arxiv ID (`arxiv:2307.09288`), or PDF URL
- `pages` (optional) — Page range like `"1-5"` or `"3"`. Omit for all pages.

### ask_document
Ask a natural language question. Returns answer with page citations.

```
ask_document(document_id: "doc-abc123", question: "What was total revenue in 2024?")
```

Parameters:
- `document_id` (required) — Document ID, arxiv ID, or PDF URL
- `question` (required) — Question to ask

Returns: Answer text + citations (page number + supporting snippet) + trace metadata.

### extract_data
Extract structured data using a prompt and JSON schema.

```
extract_data(
  document_id: "doc-abc123",
  prompt: "Extract all line items from this invoice",
  json_schema: {
    "type": "object",
    "properties": {
      "line_items": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "description": {"type": "string"},
            "quantity": {"type": "number"},
            "unit_price": {"type": "number"},
            "total": {"type": "number"}
          }
        }
      },
      "grand_total": {"type": "number"}
    }
  }
)
```

Parameters:
- `document_id` (required) — Document ID
- `prompt` (required) — What to extract
- `json_schema` (required) — JSON Schema for the output shape

### get_document_status
Check processing phase and progress.

```
get_document_status(document_id: "doc-abc123")
```

Returns: `phase` (uploading, extracting, complete, error), `page_count`, `total_nodes`.

### list_documents
List your uploaded documents.

```
list_documents(limit: 20)
```

## Common Workflows

### Extract tables from a financial report
1. `upload_document(url: "https://example.com/10k.pdf", wait: true)`
2. `extract_data(document_id: "doc-xxx", prompt: "Extract the income statement", json_schema: {...})`

### Q&A over a research paper
1. `upload_document(url: "https://arxiv.org/pdf/2307.09288")`
2. `ask_document(document_id: "doc-xxx", question: "What is the main contribution?")`

### Read specific pages
1. `read_document(document_id: "doc-xxx", pages: "1-3")` — just the intro
2. `read_document(document_id: "doc-xxx", pages: "45-50")` — financial statements

## URL Resolution

`document_id` accepts multiple formats:
- `doc-abc123` — Direct document ID
- `arxiv:2307.09288` — Arxiv paper (auto-resolved via public sources)
- `https://arxiv.org/pdf/2307.09288` — Any registered public PDF URL

## Notes

- Documents are processed asynchronously. Use `wait: true` or poll `get_document_status`.
- `read_document` output truncated at ~100k chars. Use `pages` param for large docs.
- `extract_data` works best on documents under ~100 pages.
- Page images available at `https://res.okrapdf.com/v1/documents/{id}/pg_N.png`
