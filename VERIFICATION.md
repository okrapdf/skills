# Skills Verification Log

Started 2026-04-24. Verifies that each skill in `skills/` works as the README claims.

## Method

- Read SKILL.md, identify install path + claimed inputs/outputs/scripts
- Run `npx skills add okrapdf/skills --skill <name>` to a temp dir
- Run any cookbook command/template against a small test case
- Record result + any fixes shipped

## Results

| Skill | Status | Verified | Notes |
|-------|--------|----------|-------|
| okra-cli | ⚠️ drifted | 2026-04-24 | TODO at top of SKILL.md — claims `run`, `entities`, `query`, `jobs`, `chat send`, `chat view`, `page get`, `extract --processor/--tables-only/--text-only/-d/--images` which don't exist in v0.14.0. v0.14 promoted only: `auth`, `upload`, `extract --schema`, `chat --doc`, `list`, `delete`, `read`, `collection query`. Install path (`npm install -g okrapdf`) + binary `okra` ✓. Auth via `OKRA_API_KEY` env or `okra auth set-key` ✓. |
| okra-create | ✅ works | 2026-04-24 | Worker `pdf-render-agent.steventsao.workers.dev` live. `/health` + `/browser/ping` 200. `POST /render` with `type:minimal, skipCover:true` returned valid PDF v1.4 (1849 B, 1 pg) in 463ms — matches claimed ~400ms-2s. `templates/editorial-hero.html` present. Did NOT smoke-test `/codemode/run` or `/browser/html-to-pdf` modes; canonical `/render` path is the breadth check. |
| okra-curl | ✅ works | 2026-04-24 | Smoke-tested 4 canonical endpoints — `GET /v1/vendors` (unauthed) 200, `GET /v1/documents?limit=1` 200, `GET /v1/files?limit=1` 200, `GET /v1/collections` 200. `Authorization: Bearer $OKRA_API_KEY` ✓. Fixed one drift: `/status` example was `{phase, page_count, total_nodes}` (snake) but API returns `{documentId, phase, pagesCompleted, pagesTotal, totalNodes}` (camel) — patched in SKILL.md. Did NOT smoke-test upload/chat/exports/presign — would burn credits. |
| okra-mcp | 🚫 skipped | 2026-04-24 | MCP mentions removed from README pending refresh; skill dir stays but is not advertised |
| okra-playground-share | ⏳ pending | — | — |
| okra-public-docs | ⏳ pending | — | — |
| okra-wiki | ⏳ pending | — | — |

## Cross-check vs README

After all skills are verified, ensure README skills table matches `skills/` dir contents exactly.
