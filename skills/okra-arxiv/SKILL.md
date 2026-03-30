---
name: okra-arxiv
description: Query pre-extracted arxiv AI research papers via OkraPDF MCP — read full text, ask questions, compare papers, extract methods/results. 400+ papers from cs.AI, cs.CL, cs.LG updated weekly.
---

# OkraPDF Arxiv Papers MCP

400+ recent AI research papers from arxiv, fully extracted and queryable via MCP. Covers cs.AI (Artificial Intelligence), cs.CL (Computation and Language/NLP), and cs.LG (Machine Learning).

Papers are parsed with Docling OCR on GPU — tables, equations, figures, and full text preserved as structured markdown.

## Setup

Add to `~/.claude/mcp.json` (Claude Code) or `.cursor/mcp.json` (Cursor):

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

Get a free API key at [okrapdf.com](https://okrapdf.com) (Settings > API Keys). Restart your agent after adding.

## Reading Arxiv Papers

Use the `read_document` tool with an arxiv ID:

```
# Read by arxiv ID (preferred)
read_document(document_id: "arxiv:2603.26653")

# Read by URL
read_document(document_id: "https://arxiv.org/pdf/2603.26653")

# Read specific pages (for long papers)
read_document(document_id: "arxiv:2603.26653", pages: "1-5")
```

No upload needed — papers are pre-indexed as public sources. Just pass the arxiv ID.

## Asking Questions About Papers

```
# Summarize a paper
ask_document(document_id: "arxiv:2603.26653", question: "What is the main contribution of this paper?")

# Extract methodology
ask_document(document_id: "arxiv:2603.26653", question: "Describe the model architecture and training procedure")

# Get specific results
ask_document(document_id: "arxiv:2603.26653", question: "What were the benchmark results on MMLU and HumanEval?")

# Compare with baselines
ask_document(document_id: "arxiv:2603.26653", question: "How does this compare to GPT-4 and Claude on the reported benchmarks?")
```

Returns: Answer with page citations (page number + supporting text snippet).

## Extracting Structured Data

Pull structured data from papers using JSON schemas:

```
# Extract benchmark results as a table
extract_data(
  document_id: "arxiv:2603.26653",
  prompt: "Extract all benchmark results with model names, dataset names, and scores",
  json_schema: {
    "type": "object",
    "properties": {
      "benchmarks": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "model": {"type": "string"},
            "dataset": {"type": "string"},
            "metric": {"type": "string"},
            "score": {"type": "number"}
          }
        }
      }
    }
  }
)

# Extract paper metadata
extract_data(
  document_id: "arxiv:2603.26653",
  prompt: "Extract title, authors, abstract, and keywords",
  json_schema: {
    "type": "object",
    "properties": {
      "title": {"type": "string"},
      "authors": {"type": "array", "items": {"type": "string"}},
      "abstract": {"type": "string"},
      "keywords": {"type": "array", "items": {"type": "string"}}
    }
  }
)
```

## Example Workflows

### Quick paper summary
```
ask_document(document_id: "arxiv:2603.26653", question: "Give me a 3-sentence summary")
→ "This paper introduces PerceptionComp, a benchmark for complex perceptual reasoning..."
```

### Literature survey
```
# Read abstracts from several recent agent papers
read_document(document_id: "arxiv:2603.26499", pages: "1")   # AIRA_2
read_document(document_id: "arxiv:2603.26266", pages: "1")   # GUIDE
read_document(document_id: "arxiv:2603.26005", pages: "1")   # AutoB2G

# Ask targeted questions
ask_document(document_id: "arxiv:2603.26499", question: "What bottlenecks in AI research does this address?")
ask_document(document_id: "arxiv:2603.26266", question: "How does this use video retrieval for GUI agents?")
```

### Compare methods across papers
```
# Same question, different papers
ask_document(document_id: "arxiv:2603.18272", question: "How is retrieval-augmented experience used?")
ask_document(document_id: "arxiv:2603.07379", question: "How is retrieval-augmented experience used?")
```

### Extract all tables from a paper
```
read_document(document_id: "arxiv:2603.26653")
→ Full markdown including all tables preserved as markdown tables
```

### Build a benchmark comparison spreadsheet
```
# Extract results from multiple papers
extract_data(document_id: "arxiv:2603.XXXXX", prompt: "Extract main results table", json_schema: {...})
extract_data(document_id: "arxiv:2603.YYYYY", prompt: "Extract main results table", json_schema: {...})
```

## Checking Coverage

### Browse the manifest

The full list of ingested papers is in [`papers.json`](papers.json) in this skill directory. Each entry has `id`, `title`, `categories`, and `pdf` URL.

### Discover new papers via Semantic Scholar API

Semantic Scholar indexes arxiv papers with rich metadata (abstracts, citations, authors). Use it to find papers, then check if they're in our collection.

```bash
# Search for recent papers on a topic
curl -s "https://api.semanticscholar.org/graph/v1/paper/search?query=agentic+RAG&year=2026&fields=externalIds,title,abstract,citationCount&limit=10" | jq '.data[] | {arxiv: .externalIds.ArXiv, title, citations: .citationCount}'

# Look up a specific arxiv paper
curl -s "https://api.semanticscholar.org/graph/v1/paper/ArXiv:2603.26653?fields=title,abstract,authors.name,citationCount"

# Bulk lookup — check which papers from a list are indexed
curl -s -X POST "https://api.semanticscholar.org/graph/v1/paper/batch" \
  -H "Content-Type: application/json" \
  -d '{"ids": ["ArXiv:2603.26653", "ArXiv:2603.18272"], "fields": "title,externalIds,citationCount"}'
```

No API key needed for basic use (rate limit: ~100 req/min). For higher limits, get a free key at [semanticscholar.org/product/api](https://www.semanticscholar.org/product/api).

### Discover via arxiv RSS feeds

These are the same feeds used to build this collection:

```
https://rss.arxiv.org/rss/cs.AI    # Artificial Intelligence
https://rss.arxiv.org/rss/cs.CL    # Computation and Language (NLP)
https://rss.arxiv.org/rss/cs.LG    # Machine Learning
```

Each feed returns the latest ~200 submissions. Parse with any RSS reader or:

```bash
curl -s "https://rss.arxiv.org/rss/cs.AI" | python3 -c "
import sys, xml.etree.ElementTree as ET
root = ET.fromstring(sys.stdin.read())
for item in root.findall('.//item')[:10]:
    link = item.find('link').text
    title = item.find('title').text[:80]
    aid = link.split('/abs/')[-1]
    print(f'{aid}  {title}')
"
```

### Papers With Code

[paperswithcode.com](https://paperswithcode.com) tracks papers + code repos + benchmarks. Their API is free:

```bash
# Search papers
curl -s "https://paperswithcode.com/api/v1/papers/?q=agentic+RAG&items_per_page=5" | jq '.results[] | {title, arxiv_id, abstract}'
```

## Available Papers (March 2026 Snapshot)

411 papers from the past 2 weeks across three categories:

| Category | Count | Topics |
|----------|-------|--------|
| cs.AI | ~200 | Agents, planning, multi-agent systems, reasoning, knowledge |
| cs.CL | ~100 | LLMs, NLP, prompting, RAG, text generation, dialogue |
| cs.LG | ~200 | Training methods, optimization, architectures, benchmarks |

### Notable Papers

| ID | Title | Categories |
|----|-------|-----------|
| 2603.26653 | PerceptionComp: Video Benchmark for Complex Perception-Centric Reasoning | cs.AI, cs.CL, cs.LG |
| 2603.26499 | AIRA_2: Overcoming Bottlenecks in AI Research Agents | cs.AI |
| 2603.26512 | CADSmith: Multi-Agent CAD Generation | cs.AI |
| 2603.26266 | GUIDE: Resolving Domain Bias in GUI Agents | cs.AI |
| 2603.25747 | BeSafe-Bench: Behavioral Safety Risks of Situated Agents | cs.AI |
| 2603.18272 | Retrieval-Augmented LLM Agents: Learning from Experience | cs.CL |
| 2603.07379 | SoK: Agentic RAG | cs.AI, cs.CL |
| 2603.10062 | Multi-Agent Memory from a Computer Architecture Perspective | cs.AI |

Full list: see [`papers.json`](papers.json) (411 entries with IDs, titles, categories, PDF URLs).

## How It Works

1. Papers collected from arxiv RSS feeds (cs.AI, cs.CL, cs.LG)
2. PDFs downloaded and parsed with Docling on GPU
3. Full DoclingDocument exported (markdown + structured JSON with bboxes)
4. Ingested into OkraPDF as public sources
5. Queryable via MCP using `arxiv:XXXX.XXXXX` document ID format

New papers added regularly. Collection covers the latest 2 weeks of submissions.

## Tips

- Use `arxiv:XXXX.XXXXX` format (not full URL) for cleaner queries
- `pages: "1"` is a fast way to read just the abstract/intro
- For survey papers (50+ pages), use `ask_document` instead of reading the whole thing
- `extract_data` with JSON schemas is ideal for pulling benchmark tables programmatically
- Cross-paper comparison: ask the same question to multiple papers and compare answers
- If a paper isn't found, it may not be in the current snapshot — upload it yourself with `upload_document`
- Use Semantic Scholar to find papers by topic, then query them here by arxiv ID
