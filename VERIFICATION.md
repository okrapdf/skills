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
| okra-mcp | ⚠️ drifted | 2026-04-24 | Endpoint `https://api.okrapdf.com/mcp` healthy (initialize + tools/list 200 with `Accept: application/json, text/event-stream`). Setup config in SKILL.md is correct. **Tool catalog drift:** SKILL.md claims 6 tools (`upload_document`, `read_document`, `ask_document`, `extract_data`, `get_document_status`, `list_documents`); live server returns only 3 (`upload_document` ✓, plus new `describe_collection` and `execute_code` that replace the granular flow). 5 of 6 documented tools would 404. TODO filed at top of SKILL.md. README correctly excludes this skill until rewrite. |
| okra-playground-share | ✅ works | 2026-04-24 | Smoke-tested core flow — `POST /session` 200 with seed-doc array, `GET /plugins?docId=...` 200 returning 18 variants, `GET /shares/<example token>` 200 returning snapshot. Three drift fixes applied: (1) seed catalog rotated to ParseBench slices, replaced static table with "discover via POST /session" guidance; (2) plugin catalog expanded 5→18 providers (added gemini-3.x, chandra, mineru, azure-di-layout, etc.); (3) GET /shares response shape didn't include `token`/`url` — clarified those come from POST /shares, with full live response shape. Did NOT smoke-test `/compare` or `POST /shares` — burns vendor credit. |
| okra-public-docs | ⏳ pending | — | — |
| okra-wiki | ⏳ pending | — | — |

## Cross-check vs README

After all skills are verified, ensure README skills table matches `skills/` dir contents exactly.
