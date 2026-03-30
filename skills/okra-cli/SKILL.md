---
name: okra-cli
description: OkraPDF CLI for PDF extraction, document chat, table export, entity queries, and collection management from the terminal.
---

# OkraPDF CLI

Extract PDFs, chat with documents, query tables, and manage collections from the command line.

## Install

```bash
npm install -g okrapdf
okra auth set-key YOUR_API_KEY
```

## Core Commands

### Extract a PDF

```bash
# From file
okra extract invoice.pdf

# From URL
okra extract https://arxiv.org/pdf/2307.09288

# With specific processor
okra extract report.pdf --processor llamaparse

# Output as JSON
okra extract report.pdf -o json

# One-shot: upload + extract + ask
okra run report.pdf "What was total revenue?"
```

Extract options:
- `-o json|markdown|table` — Output format
- `--processor` — `textlayer`, `llamaparse`, `unstructured`, `azure-di`, `docai`, `gemini`, `qwen`
- `--tables-only` — Only extract tables
- `--text-only` — Only extract text
- `-d <dir>` — Write agentic workspace (pages, entities, images)
- `--images` — Include page images and entity crops (requires `-d`)
- `-q, --quiet` — Minimal output (for piping)

### Document Management

```bash
# List documents
okra list

# Read document content
okra read doc-abc123

# Get document status
okra status doc-abc123

# Delete document
okra delete doc-abc123
```

### Chat with a Document

```bash
# Interactive chat
okra chat doc-abc123

# One-shot question
okra chat send doc-abc123 -m "What are the key findings?"

# View past chat session
okra chat view session-xyz
```

### Tables and Entities

```bash
# List tables in a document
okra tables doc-abc123

# Get specific table
okra tables get doc-abc123 table-0

# List all entities
okra entities list doc-abc123

# Export entity images
okra entities images doc-abc123

# jQuery-style selectors
okra query doc-abc123 "table:has(revenue)"
okra find "total revenue 2024"
```

### Page Access

```bash
# Get page content
okra page get doc-abc123 1

# Table of contents
okra toc doc-abc123

# Document tree
okra tree doc-abc123

# Search content
okra search doc-abc123 "revenue"
```

### Collections

```bash
# List collections
okra collections list

# Create collection
okra collections create --name "Q4 Reports"

# Add documents
okra collections add-docs col-xxx doc-abc123 doc-def456

# Query across collection
okra collections query col-xxx "Compare revenue growth across all companies"

# Export collection data
okra collections export col-xxx
```

### Auth

```bash
okra auth set-key YOUR_API_KEY
okra auth status
okra auth whoami
```

## Piping and Scripting

```bash
# Extract and pipe to jq
okra extract report.pdf -o json -q | jq '.entities[] | select(.type == "table")'

# Batch extract
for pdf in *.pdf; do okra extract "$pdf" -o json -q > "${pdf%.pdf}.json"; done

# Extract tables to CSV
okra tables doc-abc123 | okra query - "table[0]" -o csv
```

## Predictable URLs

Page images and exports follow deterministic URL patterns:

```
# Page image
https://res.okrapdf.com/v1/documents/{id}/pg_1.png

# With transforms
https://res.okrapdf.com/v1/documents/{id}/w_400,h_300/pg_1.png

# Exports
https://res.okrapdf.com/exports/{id}/markdown
https://res.okrapdf.com/exports/{id}/excel
https://res.okrapdf.com/exports/{id}/snapshot
```

## Available Processors

| Processor | Best For | Speed |
|-----------|----------|-------|
| textlayer | Native PDFs with selectable text | Fast |
| llamaparse | Complex layouts, mixed content | Medium |
| unstructured | General purpose | Medium |
| azure-di | Forms, invoices, receipts | Medium |
| docai | High-accuracy OCR | Slow |
| gemini | Vision-based extraction | Medium |
| qwen | Open-source VLM extraction | Medium |
