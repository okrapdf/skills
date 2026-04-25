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
| okra-cli | ⏳ pending | — | — |
| okra-create | ⏳ pending | — | — |
| okra-curl | ⏳ pending | — | — |
| okra-mcp | 🚫 skipped | 2026-04-24 | MCP mentions removed from README pending refresh; skill dir stays but is not advertised |
| okra-playground-share | ⏳ pending | — | — |
| okra-public-docs | ⏳ pending | — | — |
| okra-wiki | ⏳ pending | — | — |

## Cross-check vs README

After all skills are verified, ensure README skills table matches `skills/` dir contents exactly.
