# OkraPDF Agent Skills

Agent skills for [OkraPDF](https://okrapdf.com) — PDF extraction, document chat, structured data extraction, and programmable sandbox transforms.

## Install

```bash
# Install all skills
npx skills add okrapdf/skills --all

# Install specific skill
npx skills add okrapdf/skills --skill okra

# Install for specific agent
npx skills add okrapdf/skills -a claude-code -g
```

## Skills

| Skill | Description |
|-------|-------------|
| [okra](skills/okra/) | Upload PDFs, read content, ask questions, extract structured data, run sandbox transforms, and manage collections — via MCP, CLI, or HTTP |
| [okra-public-docs](skills/okra-public-docs/) | Pre-extracted public docs — arxiv AI papers, SEC 10-K/10-Q filings |

## Quick Start

1. Get an API key at [okrapdf.com](https://okrapdf.com)
2. Install the skill: `npx skills add okrapdf/skills --skill okra -g`
3. Add MCP config to your agent:

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

4. Ask your agent:
   - "Upload this PDF and extract all tables"
   - "Compare capex across these filings with run_sandbox"
   - "Use DOCS.querySql to pull exact evidence lines, then compute the ratio"

## Links

- [Documentation](https://docs.okrapdf.com)
- [API Reference](https://api.okrapdf.com)
- [MCP Endpoint](https://api.okrapdf.com/mcp)
- [SDK](https://github.com/okrapdf/sdk)
