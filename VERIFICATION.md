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
| okra-create | ⏳ pending | — | — |
| okra-curl | ⏳ pending | — | — |
| okra-mcp | 🚫 skipped | 2026-04-24 | MCP mentions removed from README pending refresh; skill dir stays but is not advertised |
| okra-playground-share | ⏳ pending | — | — |
| okra-public-docs | ⏳ pending | — | — |
| okra-wiki | ⏳ pending | — | — |

## Cross-check vs README

After all skills are verified, ensure README skills table matches `skills/` dir contents exactly.
