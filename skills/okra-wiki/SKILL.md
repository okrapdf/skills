---
name: okra-wiki
description: Generate a static wiki from an OkraPDF collection — export markdown, synthesize articles, build cross-references, deploy to Cloudflare Pages or any static host.
---

# OkraPDF Wiki Builder

Turn any OkraPDF collection into a navigable wiki. Export pre-extracted markdown, synthesize wiki articles with cross-references, and deploy as a static site.

## Motivation

OkraPDF already extracts PDFs into clean markdown and groups them into collections. But extracted markdown is not a wiki — it's raw source text. The gap between "I have 20 parsed PDFs" and "I have a knowledge base I can browse and share" requires synthesis: summarizing documents, categorizing them, generating cross-references, and compiling into a navigable site.

This skill bridges that gap. It uses the collection API (export, query, sandbox) as building blocks and orchestrates them into a wiki pipeline — so users go from collection to deployed wiki without building the glue themselves.

## Prerequisites

- `OKRA_API_KEY` environment variable set
- `mkdocs` installed (`pip install mkdocs mkdocs-material`) — or any static site generator
- `jq` for JSON processing

## Workflow

```
Collection (PDFs) → Export markdown → Synthesize articles → Build static site → Deploy
```

### Step 1: Export collection markdown

Pull all documents as markdown from a collection:

```bash
# Stream NDJSON export, split into per-doc markdown files
COLLECTION_ID="col-xxx"

curl -sN "https://api.okrapdf.com/v1/collections/$COLLECTION_ID/export?format=markdown" \
  -H "Authorization: Bearer $OKRA_API_KEY" \
  | jq -r 'select(.type == "document") | "\(.doc_id)\t\(.file_name)\t\(.pages | map(.content) | join("\n\n---\n\n"))"' \
  | while IFS=$'\t' read -r doc_id title content; do
      slug=$(echo "$title" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9]/-/g' | sed 's/--*/-/g' | sed 's/^-//;s/-$//')
      echo "$content" > "wiki/raw/${slug}.md"
      echo "Exported: $title → wiki/raw/${slug}.md"
    done
```

### Step 2: Get collection metadata

```bash
curl -s "https://api.okrapdf.com/v1/collections/$COLLECTION_ID" \
  -H "Authorization: Bearer $OKRA_API_KEY" \
  | jq '{name, description, doc_count: .document_count, docs: [.documents[] | {id, title: .file_name, pages: .pages_total}]}'
```

### Step 3: Synthesize wiki articles

Use the collection query endpoint to generate wiki-style summaries:

**Per-document summaries (fanout):**
```bash
curl -sN -X POST "https://api.okrapdf.com/v1/collections/$COLLECTION_ID/query" \
  -H "Authorization: Bearer $OKRA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "Write a 300-500 word wiki article summarizing this document. Include: key contributions, methods, results, and significance. Use markdown headers. End with a See Also section listing related concepts."
  }'
```

**Cross-document synthesis (sandbox mode):**
```bash
curl -s -X POST "https://api.okrapdf.com/v1/collections/$COLLECTION_ID/query" \
  -H "Authorization: Bearer $OKRA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "Analyze all documents and produce: 1) A list of 5-8 major themes/categories, 2) For each document, which categories it belongs to, 3) For each document, which other documents it most relates to and why. Return as JSON.",
    "mode": "sandbox"
  }'
```

### Step 4: Build navigation with sandbox transforms

Use the sandbox to generate a wiki index and category structure server-side:

```bash
curl -s -X POST "https://api.okrapdf.com/v1/sandbox/run" \
  -H "Authorization: Bearer $OKRA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "apiKey": "'"$OKRA_API_KEY"'",
    "code": "const docs = await DOCS.list(); const index = docs.map(d => ({ id: d.id, title: d.file_name, pages: d.total_pages, slug: d.file_name.toLowerCase().replace(/[^a-z0-9]+/g, \"-\").replace(/^-|-$/g, \"\") })); return { wiki_name: \"My Wiki\", articles: index, generated: new Date().toISOString() };"
  }' | jq '.result'
```

### Step 5: Generate mkdocs site

Write an `mkdocs.yml` from the collection metadata:

```bash
WIKI_NAME=$(curl -s "https://api.okrapdf.com/v1/collections/$COLLECTION_ID" \
  -H "Authorization: Bearer $OKRA_API_KEY" | jq -r '.name')

cat > mkdocs.yml << EOF
site_name: "$WIKI_NAME"
theme:
  name: material
  palette:
    scheme: default
nav:
  - Home: index.md
EOF

# Append each article to nav
for f in wiki/*.md; do
  title=$(head -1 "$f" | sed 's/^# //')
  echo "  - \"$title\": $(basename $f)" >> mkdocs.yml
done

mkdocs build --clean
```

### Step 6: Deploy

```bash
# Cloudflare Pages
npx wrangler pages deploy site/ --project-name my-wiki --branch main

# Netlify
npx netlify deploy --dir site/ --prod

# GitHub Pages
gh-pages -d site/

# Or just serve locally
mkdocs serve
```

## Per-document chat completions

For richer wiki articles, use the OpenAI-compatible completion endpoint to have a multi-turn conversation with each document:

```bash
DOC_ID="doc-abc123"

curl -s -X POST "https://api.okrapdf.com/document/$DOC_ID/chat/completions" \
  -H "Authorization: Bearer $OKRA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [
      {"role": "system", "content": "You are a wiki editor. Write concise, factual articles with headers, cross-references in [[wikilink]] format, and a See Also section."},
      {"role": "user", "content": "Write a wiki article about this document."}
    ],
    "stream": false
  }' | jq -r '.choices[0].message.content'
```

## Structured extraction for wiki metadata

Extract structured metadata from each document to power categories, tags, and cross-references:

```bash
curl -s -X POST "https://api.okrapdf.com/document/$DOC_ID/chat/completions" \
  -H "Authorization: Bearer $OKRA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [{"role": "user", "content": "Extract metadata for this document."}],
    "response_format": {
      "type": "json_schema",
      "json_schema": {
        "name": "wiki_metadata",
        "schema": {
          "type": "object",
          "properties": {
            "title": {"type": "string"},
            "summary": {"type": "string", "description": "2-3 sentence summary"},
            "categories": {"type": "array", "items": {"type": "string"}},
            "key_concepts": {"type": "array", "items": {"type": "string"}},
            "related_topics": {"type": "array", "items": {"type": "string"}},
            "authors": {"type": "array", "items": {"type": "string"}},
            "year": {"type": "integer"}
          },
          "required": ["title", "summary", "categories", "key_concepts"]
        }
      }
    },
    "stream": false
  }' | jq '.choices[0].message.content | fromjson'
```

## Full pipeline script

Combine all steps into one script. Requires `OKRA_API_KEY`, `jq`, `mkdocs`:

```bash
#!/usr/bin/env bash
set -euo pipefail

COLLECTION_ID="${1:?Usage: okra-wiki.sh <collection-id>}"
OUT_DIR="${2:-wiki-out}"
mkdir -p "$OUT_DIR/docs"

echo "==> Fetching collection metadata..."
META=$(curl -s "https://api.okrapdf.com/v1/collections/$COLLECTION_ID" \
  -H "Authorization: Bearer $OKRA_API_KEY")
WIKI_NAME=$(echo "$META" | jq -r '.name')
DOC_IDS=$(echo "$META" | jq -r '.documents[].id')
echo "    $WIKI_NAME ($(echo "$META" | jq '.document_count') docs)"

echo "==> Generating wiki articles..."
for doc_id in $DOC_IDS; do
  ARTICLE=$(curl -s -X POST "https://api.okrapdf.com/document/$doc_id/chat/completions" \
    -H "Authorization: Bearer $OKRA_API_KEY" \
    -H "Content-Type: application/json" \
    -d '{
      "messages": [
        {"role": "system", "content": "Write a concise wiki article about this document. Use markdown headers. Include key contributions, methods, and significance. Keep under 800 words."},
        {"role": "user", "content": "Write the article."}
      ],
      "stream": false
    }' | jq -r '.choices[0].message.content // empty')

  if [ -n "$ARTICLE" ]; then
    TITLE=$(echo "$ARTICLE" | head -1 | sed 's/^#* *//')
    SLUG=$(echo "$TITLE" | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9]/-/g;s/--*/-/g;s/^-//;s/-$//')
    echo "$ARTICLE" > "$OUT_DIR/docs/${SLUG}.md"
    echo "    ✓ $TITLE"
  fi
done

echo "==> Building index..."
cat > "$OUT_DIR/docs/index.md" << EOF
# $WIKI_NAME

Generated from [OkraPDF collection]($( echo "$META" | jq -r '.public_url // "https://okrapdf.com"')).

## Articles

EOF
for f in "$OUT_DIR"/docs/*.md; do
  [ "$(basename $f)" = "index.md" ] && continue
  TITLE=$(head -1 "$f" | sed 's/^#* *//')
  echo "- [$TITLE]($(basename $f))" >> "$OUT_DIR/docs/index.md"
done

echo "==> Writing mkdocs.yml..."
cat > "$OUT_DIR/mkdocs.yml" << EOF
site_name: "$WIKI_NAME"
docs_dir: docs
site_dir: site
theme:
  name: material
  palette:
    scheme: default
EOF

echo "==> Building site..."
cd "$OUT_DIR" && mkdocs build --clean
echo "==> Done! Site at $OUT_DIR/site/"
echo "    Preview: cd $OUT_DIR && mkdocs serve"
```

## Tips

- **Start small** — test with a 3-5 doc collection before running on 50+ docs
- **Sandbox for structure, completions for content** — use sandbox transforms to build nav/categories, completion endpoint for article prose
- **Cache exports** — collection export is deterministic per document version; save to disk and re-run synthesis only
- **Wikilinks** — ask the LLM to use `[[Article Name]]` syntax, then resolve to relative paths in a post-processing step
- **Incremental updates** — track doc IDs you've already processed; only generate articles for new additions
