# Roadmap — Deferred Skills

Skills removed from the public surface on 2026-04-24 in favor of an API-first JTBD focus. Each will return when its prerequisite lands.

## API-first kept (current `skills/`)

- **okra-curl** — `curl` cookbook for completions / extraction / exports / files. The canonical "I want a PDF API" entry point.
- **okra-create** — `pdf-render-agent` Worker (baked / browser-HTML / codemode). The canonical "I want to make a PDF from an agent" entry point.

## Deferred

### okra-cli
**Why deferred:** CLI v0.14.0 ships a deliberately narrow promoted surface — `auth`, `upload`, `extract --schema`, `chat --doc`, `list`, `read`, `delete`, `collection query`. Five categories the previous SKILL.md documented (`run`, `entities`, `query`, `jobs`, `chat send/view`) plus a half-dozen `extract` flags are hidden during the clean-house RC.
**Un-defer when:** `okrapdf` CLI promotes the advanced inspection surface (post-v0.14.x clean-house). Rewrite the SKILL.md against whatever the v0.14+ help output actually exposes.
**Tracking:** see [`okrapdf/cli`](https://github.com/okrapdf/cli) milestone `cli/v0.14.0-rc.1` and successors.

### okra-mcp
**Why deferred:** Tool-catalog drift. Previous SKILL.md documented six tools (`upload_document`, `read_document`, `ask_document`, `extract_data`, `get_document_status`, `list_documents`); live `tools/list` returns three (`upload_document`, `describe_collection`, `execute_code`). The new `execute_code` surface (JS over a `DOCS` API with SQLite + FTS) is more powerful but completely undocumented and replaces the granular flow.
**Un-defer when:** rewrite around `execute_code` + `describe_collection`. New examples should lead with SQL/FTS over `nodes` and `nodes_fts`, falling back to `ask`/`extract` only when the answer needs LLM reasoning.

### okra-playground-share
**Why deferred:** Works end-to-end (`POST /session` → `GET /plugins` → `GET /compare` → `POST /shares`), but it's a share/viral surface, not an API-first JTBD. The 18-provider plugin catalog also rotates fast enough that documenting it here would re-drift quickly.
**Un-defer when:** the playground stabilizes into a stable v1 surface AND we have evidence the share-link primitive is a JTBD users hire skills for (not just a marketing surface).

### okra-public-docs
**Why deferred:** Both integration paths are broken. (1) Auth MCP at `api.okrapdf.com/mcp` is healthy but missing the `read_document`/`ask_document`/`extract_data` tools the arxiv examples used (catalog drift, see okra-mcp). (2) Zero-auth SEC MCP at `mcp.okrapdf.com/mcp` returns 404 "App 'mcp' not found" — endpoint dead. Probed alternates (`sec.okrapdf.com/mcp`, `api.okrapdf.com/sec/mcp`, `api.okrapdf.com/v1/sec/mcp`) all 404 or auth-required.
**Un-defer when:** (a) okra-mcp rewrite lands so arxiv path can use `execute_code` with `public_doc_ids`, AND (b) the zero-auth SEC MCP host is restored or the SEC tool surface is folded into the auth MCP.
**Static asset preserved:** the 411-paper arxiv manifest (`papers.json`, ~3400 lines) was useful and will be re-included when the skill returns.

### okra-wiki
**Why deferred:** Works end-to-end against the live API (collection get/export + sandbox), but it's an orchestration layer on top of the API — not the API itself. The pipeline (export → synthesize → mkdocs → deploy) is a six-step shell script and three vendors deep. Doesn't fit an API-first JTBD lens.
**Un-defer when:** there's repeat demand for "turn my collection into a wiki" as a discrete user journey AND the synthesis step is owned by an Okra primitive (not orchestrated externally with arbitrary mkdocs/Cloudflare Pages glue).

## History

- 2026-04-24 — README claims audited; 4 inline drift fixes shipped, 3 TODOs filed at top of SKILL.md, then 5 skills removed in favor of API-first JTBD focus.
- 2026-04-24 — MCP mentions removed from README (then full okra-mcp deletion).

See `git log -- skills/` for the audit trail of each removed skill.
