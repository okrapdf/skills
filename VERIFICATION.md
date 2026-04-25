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
| okra-public-docs | ⚠️ drifted | 2026-04-24 | `papers.json` (3400-line arxiv manifest) ✓. **Both MCP paths broken/drifted:** (1) auth MCP `api.okrapdf.com/mcp` healthy but missing `read_document`/`ask_document`/`extract_data` tools used in arxiv examples (catalog drift, see okra-mcp); (2) zero-auth SEC MCP `mcp.okrapdf.com/mcp` returns 404 "App 'mcp' not found" — endpoint is DEAD. Probed alternates: `sec.okrapdf.com/mcp` 404, `api.okrapdf.com/sec/mcp` 404, `api.okrapdf.com/v1/sec/mcp` 401 (exists but auth-required, not zero-auth). TODO filed. |
| okra-wiki | ✅ works | 2026-04-24 | Smoke-tested 3 building blocks: `GET /v1/collections/<id>` 200 with all documented keys (`name`, `documents`, `document_count`, etc.), `GET /v1/collections/<id>/export?format=markdown` 200 streaming NDJSON `{type:"start"}` + `{type:"result", doc_id, file_name, pages:[{pageNumber, content}]}` (matches SKILL.md jq filter), `POST /v1/sandbox/run` 200 with `apiKey + code` returning `{ok, result, logs, runId, run:{id, status, publish, get, delete, embedUrl}}` and `await DOCS.list()` works inside (returned 100 docs). Did NOT smoke-test `/collections/<id>/query` or `/document/<id>/chat/completions` — LLM burn; same endpoints already verified via okra-curl. |

## 2026-04-24 — surface reduced to API-first JTBDs

After the audit below completed, the surface was pruned to two skills (`okra-curl`, `okra-create`). The other five (`okra-cli`, `okra-mcp`, `okra-playground-share`, `okra-public-docs`, `okra-wiki`) were deleted from `skills/` and moved to `ROADMAP.md` with explicit un-defer prerequisites. The audit notes below are kept as the historical reason each was removed.

## Cross-check vs README (2026-04-24, pre-prune)

`skills/` directory held 7 skills; README advertised 6:

| Skill | In `skills/` | In README | OK? |
|---|:---:|:---:|---|
| okra-cli | ✓ | ✓ | sync |
| okra-create | ✓ | ✓ | sync |
| okra-curl | ✓ | ✓ | sync |
| okra-mcp | ✓ | — | intentionally hidden (TODO at top of SKILL.md, until tool-catalog rewrite) |
| okra-playground-share | ✓ | ✓ | sync |
| okra-public-docs | ✓ | ✓ | **questionable** — README advertises it but both MCP paths are broken/drifted (see TODO at top of SKILL.md). Consider hiding from README until arxiv path rewritten against `execute_code` and SEC MCP host restored, mirroring the okra-mcp treatment. |
| okra-wiki | ✓ | ✓ | sync |

## Summary

| Status | Count | Skills |
|---|---:|---|
| ✅ works | 4 | okra-create, okra-curl (1 fix), okra-playground-share (3 fixes), okra-wiki |
| ⚠️ drifted (TODO filed) | 3 | okra-cli, okra-mcp, okra-public-docs |

**Net fixes shipped:** 4 inline drift patches (okra-curl status shape; okra-playground-share seed catalog, plugin list, GET /shares response). **3 TODOs filed at top of SKILL.md** for skills whose drift is too large for one iteration: okra-cli (CLI v0.14.0 hides advanced commands), okra-mcp (5 of 6 documented tools missing on live server), okra-public-docs (auth-MCP catalog drift + zero-auth SEC MCP host dead).

**Outstanding decision:** whether to also hide `okra-public-docs` from the README until its TODO clears (same treatment as `okra-mcp`).
