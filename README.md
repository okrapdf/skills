# OkraPDF Agent Skills

Agent skills for [OkraPDF](https://okrapdf.com) — PDF extraction, document chat, structured data extraction, and wiki generation.

OkraPDF turns PDFs into structured, queryable document objects. These skills teach AI agents how to use those objects — so you can say "extract all tables from this 10-K" or "turn this paper collection into a wiki" and the agent knows the right API calls, patterns, and trade-offs.

## Install

```bash
# Install all skills
npx skills add okrapdf/skills --all

# Install specific skill
npx skills add okrapdf/skills --skill okra-mcp

# Install for specific agent
npx skills add okrapdf/skills -a claude-code -g
```

## Skills

| Skill | Description |
|-------|-------------|
| [okra-mcp](skills/okra-mcp/) | Connect to OkraPDF via MCP — upload, read, ask, extract |
| [okra-cli](skills/okra-cli/) | CLI-based PDF extraction, document chat, and collections |
| [okra-curl](skills/okra-curl/) | HTTP/curl cookbook for the OkraPDF REST API, including passive file assets |
| [okra-playground-share](skills/okra-playground-share/) | Create shareable `pgs_...` playground links — upload a PDF, run plugins, publish an immutable snapshot |
| [okra-public-docs](skills/okra-public-docs/) | Pre-extracted public docs — arxiv papers, SEC filings |
| [okra-wiki](skills/okra-wiki/) | Generate a static wiki from a collection — export, synthesize, deploy |

## Quick Start

1. Get an API key at [okrapdf.com](https://okrapdf.com)
2. Install the skill: `npx skills add okrapdf/skills --skill okra-mcp -g`
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

4. Ask your agent: "Upload this PDF and extract all tables"

## Links

- [Documentation](https://docs.okrapdf.com)
- [API Reference](https://api.okrapdf.com)
- [MCP Endpoint](https://api.okrapdf.com/mcp)
- [SDK](https://github.com/okrapdf/sdk)
