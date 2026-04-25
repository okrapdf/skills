# OkraPDF Agent Skills

Agent skills for the [OkraPDF API](https://api.okrapdf.com). Two JTBDs, both API-first.

## Skills

| JTBD | Skill | What it teaches |
|------|-------|-----------------|
| Get a completion endpoint on a PDF, fast | [okra-curl](skills/okra-curl/) | `curl` cookbook for `POST /v1/documents` → OpenAI-compatible `POST /document/{id}/chat/completions`, structured extraction with `response_format`, exports, page images. The fastest path from "I have a PDF" to "I have an API." |
| Generate a designed PDF from an agent | [okra-create](skills/okra-create/) | `POST /render` (baked design system) + `POST /browser/html-to-pdf` (custom HTML cover) + `POST /codemode/run` (composed). One Worker, three modes. |

## Install

```bash
# Both skills
npx skills add okrapdf/skills --all

# Just one
npx skills add okrapdf/skills --skill okra-curl
npx skills add okrapdf/skills --skill okra-create

# For a specific agent (Claude Code, Cursor, OpenCode...)
npx skills add okrapdf/skills -a claude-code -g
```

## Quick start

1. Get an API key at [okrapdf.com](https://okrapdf.com)
2. `export OKRA_API_KEY=okra_...`
3. Pick a JTBD above; the SKILL.md has copy-pasteable curl examples.

## Deferred

The full agent surface (CLI wrappers, MCP integration, playground share links, public-docs corpora, wiki generation) is roadmapped — see [`ROADMAP.md`](ROADMAP.md). API-first comes first; agent ergonomics layer on after.

## Links

- [Documentation](https://docs.okrapdf.com)
- [API Reference](https://api.okrapdf.com)
- [SDK](https://github.com/okrapdf/sdk)
