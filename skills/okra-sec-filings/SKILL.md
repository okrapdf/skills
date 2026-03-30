---
name: okra-sec-filings
description: Zero-auth MCP server for public SEC 10-K and 10-Q filings — browse filings, read extracted content, ask questions across companies.
---

# OkraPDF SEC Filings MCP

Pre-extracted SEC 10-K and 10-Q filings accessible via MCP. No API key needed — all filings are public.

## Setup

```json
{
  "mcpServers": {
    "okra-sec": {
      "type": "url",
      "url": "https://mcp.okrapdf.com/mcp"
    }
  }
}
```

No auth required. Public SEC filings, free to query.

## Available Tools

### read_filing_index
Browse available SEC filings. Filter by ticker or filing type.

```
read_filing_index()
read_filing_index(ticker: "NVDA")
read_filing_index(ticker: "AAPL", filing_type: "10-K")
read_filing_index(limit: 50)
```

Returns: ticker, company name, filing type, period, page count, summary.

### read_filing_contents
Get full extracted content from a specific filing as markdown.

```
read_filing_contents(ticker: "NVDA", filing: "10-k-2024")
read_filing_contents(ticker: "MSFT", filing: "10-Q/2024")
```

Filing formats accepted: `10-k-2024`, `10-K/2024`, `2024-10K`

Returns: Full markdown with tables, text, page markers.

### ask_question
AI Q&A over filings. Supports cross-company analysis (up to 10 tickers).

```
# Single company
ask_question(question: "What was NVIDIA's data center revenue in FY2024?", tickers: ["NVDA"])

# Cross-company comparison
ask_question(
  question: "Compare R&D spending as % of revenue",
  tickers: ["AAPL", "MSFT", "GOOGL", "NVDA", "META", "AMZN", "TSLA"]
)

# Specific filing
ask_question(
  question: "What are the key risk factors?",
  tickers: ["TSLA"],
  filing: "10-k-2024"
)
```

### get_verification_summary
Check extraction quality for a filing. Shows which pages are verified, need review, or failed.

```
get_verification_summary(document_id: "doc-abc123")
get_verification_summary(document_id: "doc-abc123", status: "needs_review")
get_verification_summary(document_id: "doc-abc123", below: 0.8)
```

### verify_pages
Approve or flag pages in a filing for quality control.

```
verify_pages(document_id: "doc-abc123", action: "approve", pages: [1, 2, 3])
verify_pages(document_id: "doc-abc123", action: "approve", confidence_above: 0.9)
verify_pages(document_id: "doc-abc123", action: "flag", pages: [45], reason: "Table misaligned")
```

## Example Workflows

### Financial analysis
1. `read_filing_index(ticker: "NVDA")` — see available filings
2. `read_filing_contents(ticker: "NVDA", filing: "10-k-2024")` — get full text
3. `ask_question(question: "What drove revenue growth?", tickers: ["NVDA"])` — AI answer with citations

### Cross-company comparison
1. `ask_question(question: "Compare gross margins", tickers: ["AAPL", "MSFT", "GOOGL"])` — multi-doc synthesis

### Quality audit
1. `get_verification_summary(document_id: "doc-xxx")` — check extraction quality
2. `verify_pages(document_id: "doc-xxx", action: "approve", confidence_above: 0.95)` — bulk approve high-confidence pages

## Available Companies

Filings from major US public companies including Magnificent 7 (AAPL, MSFT, GOOGL, AMZN, NVDA, META, TSLA) and FinanceBench dataset companies. Use `read_filing_index()` to browse the full list.
