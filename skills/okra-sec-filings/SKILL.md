---
name: okra-sec-filings
description: Zero-auth MCP server for public SEC 10-K and 10-Q filings — browse filings, read extracted content, ask questions across companies, verify extraction quality. No signup needed.
---

# OkraPDF SEC Filings MCP

Pre-extracted SEC 10-K and 10-Q filings accessible via MCP. No API key, no signup, completely free. Every filing is fully parsed with tables, figures, and structured text.

## Setup

Add to `~/.claude/mcp.json` (Claude Code) or `.cursor/mcp.json` (Cursor):

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

No API key, no auth header, no signup. Restart your agent after adding.

**Verify:** Ask your agent "List SEC filings for NVDA." If it returns filings, you're connected.

## Available Tools

### read_filing_index
Browse all available filings. Filter by ticker, type, or both.

```
# See everything
read_filing_index()

# Filter by company
read_filing_index(ticker: "NVDA")

# Filter by type
read_filing_index(filing_type: "10-K")

# Combine filters
read_filing_index(ticker: "AAPL", filing_type: "10-K", limit: 50)
```

Returns per filing: ticker, company_name, filingType, period (e.g. "FY2024"), pageCount, publishedAt, summary.

**Use this first** to discover what's available before reading or querying.

### read_filing_contents
Get the full extracted text of a specific filing. Returns markdown with tables, text, and `<!-- Page N -->` markers.

```
read_filing_contents(ticker: "NVDA", filing: "10-k-2024")
read_filing_contents(ticker: "MSFT", filing: "10-Q/2024")
read_filing_contents(ticker: "AAPL", filing: "10-k-2023")
```

**Filing slug formats** (all equivalent):
- `10-k-2024` (lowercase, hyphenated)
- `10-K/2024` (type/year)
- `2024-10K` (year-type)

The response includes all pages as structured markdown. Tables are preserved as markdown tables. For very large filings (200+ pages), the response may be truncated — focus on specific sections by searching within the content.

### ask_question
AI-powered Q&A with grounded citations. The key tool — ask anything about any filing.

**Single company:**
```
ask_question(question: "What was NVIDIA's data center revenue?", tickers: ["NVDA"])
ask_question(question: "List all risk factors related to AI regulation", tickers: ["MSFT"])
ask_question(question: "What are the outstanding debt obligations?", tickers: ["TSLA"], filing: "10-k-2024")
```

**Cross-company (up to 10 tickers):**
```
ask_question(
  question: "Compare R&D spending as a percentage of revenue",
  tickers: ["AAPL", "MSFT", "GOOGL", "NVDA", "META", "AMZN", "TSLA"]
)

ask_question(
  question: "Which company has the highest gross margin?",
  tickers: ["AAPL", "MSFT", "GOOGL"]
)

ask_question(
  question: "Summarize each company's AI strategy",
  tickers: ["NVDA", "AMD", "INTC"]
)
```

**How it works:**
- Single ticker: queries one filing directly, returns answer with page citations
- Multiple tickers: fans out to each company's filing in parallel, then synthesizes a cross-company answer
- If a filing is missing for a ticker, the response includes a warning (doesn't fail)
- Defaults to latest 10-K when `filing` param is omitted

### get_verification_summary
Check extraction quality page-by-page. Useful for auditing before using data in production.

```
# Full summary
get_verification_summary(document_id: "doc-abc123")

# Only pages needing review
get_verification_summary(document_id: "doc-abc123", status: "needs_review")

# Pages with low confidence
get_verification_summary(document_id: "doc-abc123", below: 0.8)

# Single page detail
get_verification_summary(document_id: "doc-abc123", page: 45)
```

Returns: per-page verification status (verified, needs_review, failed, clean) and confidence scores.

**Note:** You need the document_id (not ticker) for verification tools. Get it from `read_filing_index` response or `ask_question` trace metadata.

### verify_pages
Approve or flag pages after reviewing. Use for quality control workflows.

```
# Approve specific pages
verify_pages(document_id: "doc-abc123", action: "approve", pages: [1, 2, 3])

# Bulk approve high-confidence pages
verify_pages(document_id: "doc-abc123", action: "approve", confidence_above: 0.95)

# Flag a problematic page with reason
verify_pages(document_id: "doc-abc123", action: "flag", pages: [45], reason: "Revenue table missing column headers")
```

## Example Workflows

### Quick company research
```
read_filing_index(ticker: "NVDA")
→ See available filings (10-K 2024, 10-K 2023, 10-Q Q3 2024, ...)

ask_question(question: "What drove revenue growth in FY2024?", tickers: ["NVDA"])
→ "Data center revenue grew 217% to $47.5B, driven by... [Page 38]"
```

### Cross-company competitive analysis
```
ask_question(
  question: "Compare capital expenditure and free cash flow for FY2024",
  tickers: ["AAPL", "MSFT", "GOOGL", "AMZN", "META"]
)
→ Table comparing CapEx, FCF, CapEx/Revenue ratio across all 5 companies with citations
```

### Deep dive into a specific filing
```
read_filing_contents(ticker: "TSLA", filing: "10-k-2024")
→ Full 200+ page filing as markdown

ask_question(question: "What are all the legal proceedings mentioned?", tickers: ["TSLA"], filing: "10-k-2024")
→ Extracted list of legal proceedings with page references
```

### Year-over-year analysis
```
ask_question(question: "How did gross margin change from FY2023 to FY2024?", tickers: ["NVDA"])
→ Agent queries both filings and compares
```

### Extraction quality audit
```
get_verification_summary(document_id: "doc-xxx")
→ 180/200 pages verified, 15 needs_review, 5 failed

get_verification_summary(document_id: "doc-xxx", status: "needs_review")
→ Pages 45, 67, 89, ... (usually complex tables)

verify_pages(document_id: "doc-xxx", action: "approve", confidence_above: 0.9)
→ Bulk approve 170 pages

verify_pages(document_id: "doc-xxx", action: "flag", pages: [67], reason: "Revenue breakdown table has merged cells")
```

## Available Companies

**Magnificent 7:** AAPL, MSFT, GOOGL, AMZN, NVDA, META, TSLA

**FinanceBench dataset:** ~80 companies including major banks (JPM, BAC, GS), pharma (PFE, JNJ, ABBV), industrials (GE, MMM, CAT), and more.

Use `read_filing_index()` to browse the full catalog. New filings are added as they're published.

## Tips

- Start with `read_filing_index` to discover what's available before querying
- `ask_question` with multiple tickers is the fastest way to do comparisons — no need to read each filing manually
- Filing slugs are flexible — `10-k-2024`, `10-K/2024`, and `2024-10K` all work
- For very specific data points (exact table cells), use `read_filing_contents` and search within the markdown
- Cross-company queries work best with clear, quantitative questions ("Compare X metric")
- The verification tools require document_id, not ticker — get it from other tool responses
