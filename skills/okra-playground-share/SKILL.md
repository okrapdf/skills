---
name: okra-playground-share
description: Create shareable `pgs_...` playground links from the OkraPDF playground API — upload a PDF (or reuse a seed doc), run built-in vendor plugins or custom GitHub plugins per variant, and publish an immutable snapshot anyone can open at `okrapdf.com/playground/share/pgs_...`.
---

# OkraPDF Playground Shares

Publish a PDF + extraction snapshot as a public URL like
`https://okrapdf.com/playground/share/pgs_w3wfl5zsnnt2iuJpFspC8a_8s-J-7l5c`.

No API key. Anonymous session cookie. Pick built-in vendor plugins
(`textlayer`, `llamaparse-*`, `unstructured`, `mistral-ocr-latest`, …) or
point at a plugin repo on GitHub. The result is an immutable JSON snapshot
in R2 served by the public share page.

## Base URL

```
https://okrapdf.com/api/playground
```

Proxied to `https://api.okrapdf.com/v1/playground/*`. No `Authorization`
header. The session cookie (`okra_playground`) must travel on every call.

## 60-second flow (verified)

```bash
BASE="https://okrapdf.com/api/playground"
JAR=$(mktemp)

# 1. session
curl -s -c "$JAR" -b "$JAR" -X POST "$BASE/session" \
  -H 'content-type: application/json' -d '{}'

# 2. seed doc (no upload needed for demos)
DOC=doc-0552aef6712b412782ade6ee1788d0b9   # 3M 10-K excerpt, 11 pages

# 3. discover which facets this session+doc can run
curl -s -b "$JAR" "$BASE/plugins" | jq '.plugins | map(.id)'
# → ["llamaparse-fast","llamaparse-premium","llamaparse-agentic"]

# 4. run ONE facet per compare call (critical — see gotcha below)
curl -s -b "$JAR" -c "$JAR" "$BASE/compare?docId=$DOC&variants=llamaparse-fast" \
  | jq '.variants[0] | {id, ok, nodes, elapsedMs, costUsd}'

# 5. publish share
curl -s -b "$JAR" -c "$JAR" -X POST "$BASE/shares" \
  -H 'content-type: application/json' \
  -d "{\"docId\":\"$DOC\",\"facetIds\":[\"llamaparse-fast\"],\"expiresInSeconds\":604800}" \
  | jq
```

Real response from this exact flow:
```json
{
  "token": "pgs_w3wfl5zsnnt2iuJpFspC8a_8s-J-7l5c",
  "url": "https://okrapdf.com/playground/share/pgs_w3wfl5zsnnt2iuJpFspC8a_8s-J-7l5c",
  "expiresAt": "2026-04-28T00:48:46.797Z"
}
```

The returned `url` is shareable as-is. Open it in any browser — no cookie,
no auth. Cached 60s at the edge.

## Step 1 — Session bootstrap

```bash
curl -X POST https://okrapdf.com/api/playground/session \
  -H 'content-type: application/json' -d '{}'
```

Response: `{ "sessionId": "<uuid>", "expiresAt": <ms> }`

Sets an `okra_playground` cookie (httpOnly, 1h). Send it on every
subsequent call. Pass `{ "sessionId": "<existing>" }` to rehydrate a
known session instead of minting a new one.

To reset (e.g. after hitting the cost cap):
```bash
curl -X DELETE https://okrapdf.com/api/playground/session \
  -b cookies.txt -c cookies.txt
```

## Step 2 — Provide a document

**Option A — use a built-in seed doc.** Zero setup. The catalog is rotated
periodically by the server. Always discover the current set from the
`POST /session` response — the response includes a `documents` array of
seed docs with `doc_id`, `file_name`, `pages_total`, and `description`.

```bash
curl -s -X POST https://okrapdf.com/api/playground/session \
  -H 'content-type: application/json' -d '{}' \
  | jq '.documents | map({doc_id, file_name, pages_total})'
```

As of 2026-04 the seeds are ParseBench slices (Apple, Goldman Sachs,
Home Depot, Coca-Cola, ExxonMobil 10-Ks plus OECD/IEA reports). Older
docIds like `doc-0552aef6712b412782ade6ee1788d0b9` (3M 10-K excerpt)
may still resolve directly, but rely on the session response for the
authoritative current catalog.

Seed docs auto-resolve — no `/upload` step, just pass the docId into
`/compare` and `/shares`.

**Option B — upload a PDF (multipart):**
```bash
curl -X POST https://okrapdf.com/api/playground/upload \
  -b cookies.txt -c cookies.txt \
  -F "file=@./paper.pdf"
```
Response: `{ "docId": "doc-...", "fileName": "paper.pdf" }`

Constraints:
- `application/pdf` only, ≤ ~25 MB
- **one uploaded doc per session** — `DELETE /session` and re-init to upload another

**Option C — direct-to-R2 presign (large files):**
```bash
# get presigned PUT URL
curl -X POST https://okrapdf.com/api/playground/upload/presign \
  -b cookies.txt -c cookies.txt \
  -H 'content-type: application/json' \
  -d '{"fileName":"big.pdf","contentType":"application/pdf"}'

# PUT bytes to the returned URL, then finalize
curl -X POST https://okrapdf.com/api/playground/upload/process \
  -b cookies.txt -c cookies.txt \
  -H 'content-type: application/json' \
  -d '{"uploadId":"..."}'
```

## Step 3 — Discover available facets

**Always list plugins before running compare.** The default facet map
depends on doc source — seed docs often expose only `llamaparse-*`,
uploaded PDFs get `textlayer` + more. **Unknown variant IDs silently
fall back to defaults**, so a typo like `variants=texlayer` will
happily run the wrong vendor.

```bash
curl -s https://okrapdf.com/api/playground/plugins -b cookies.txt \
  | jq '.plugins | map({id, label})'
```

## Step 4 — (Optional) Install a custom plugin from GitHub

Preview the manifest, then install. Returns a `facetId` you can feed
into `compare` and `shares` alongside built-ins.

```bash
# Preview
curl -X POST https://okrapdf.com/api/playground/plugins/github/preview \
  -b cookies.txt -c cookies.txt \
  -H 'content-type: application/json' \
  -d '{"url":"https://github.com/acme/my-parser/tree/main"}'

# Install (paste the preview object back in)
curl -X POST https://okrapdf.com/api/playground/plugins/github/install \
  -b cookies.txt -c cookies.txt \
  -H 'content-type: application/json' \
  -d '{"preview": <preview from step above>}'
# → { "plugin": { "id": "github-...", "installId": "..." } }
```

Use `installId` (or `id`) as a `facetId` in the next steps.

## Step 5 — Run each facet (one per call)

```bash
curl "https://okrapdf.com/api/playground/compare?docId=$DOC&variants=llamaparse-fast" \
  -b cookies.txt -c cookies.txt
```

Response:
```json
{
  "documentId": "doc-...",
  "fileName": "paper.pdf",
  "variants": [
    { "id": "llamaparse-fast", "ok": true, "nodes": 119, "elapsedMs": 3859, "costUsd": 0.04125, "error": null }
  ]
}
```

### ⚠️ One variant per call

`/compare` is keyed by the *set* of variants you pass. If you run
`variants=a,b,c` in one call, the snapshot is stored under the triple
key `[a,b,c]`. Later, `POST /shares` looks up each facet by its single
key `[facetId]` and 409s with `"Facet has no completed snapshot"`.

Run each variant as a separate GET, parallelize with `Promise.all`. This
is what the reference runner (`scripts/playground/run.ts`) does.

Only `ok: true` variants are shareable.

## Step 6 — Publish the share

```bash
curl -X POST https://okrapdf.com/api/playground/shares \
  -b cookies.txt -c cookies.txt \
  -H 'content-type: application/json' \
  -d '{
    "docId": "doc-...",
    "facetIds": ["llamaparse-fast", "llamaparse-premium"],
    "expiresInSeconds": 604800
  }'
```

Response:
```json
{
  "token": "pgs_w3wfl5zsnnt2iuJpFspC8a_8s-J-7l5c",
  "url": "https://okrapdf.com/playground/share/pgs_w3wfl5zsnnt2iuJpFspC8a_8s-J-7l5c",
  "expiresAt": "2026-04-28T00:48:46.797Z"
}
```

Rules:
- `facetIds` must be non-empty; each needs a `ok: true` single-variant
  snapshot from step 5 — otherwise 409.
- `expiresInSeconds` must be one of **86400 (1d) / 604800 (7d) / 2592000 (30d)**.
- `includeConversation: true` → 400. Chat is not shareable via this API.
- Tokens are immutable. To change anything, create a new share.

## Step 7 — Read a share (public, no cookie)

```bash
curl https://okrapdf.com/api/playground/shares/pgs_w3wfl5zsnnt2iuJpFspC8a_8s-J-7l5c
```

Returns the full snapshot. Response shape (verified 2026-04):
```jsonc
{
  "version": 1,
  "createdAt": "2026-04-21T...",
  "creatorSessionId": "...",
  "docId": "doc-...",
  "docSource": "seed",
  "fileName": "...",
  "pagesTotal": 11,
  "facets": [ /* per-variant nodes/cost/elapsed/error */ ],
  "totalCostUsd": 0.04125,
  "elapsedMs": 3859,
  "expiresAt": "2026-04-28T..."
}
```

Note: `token` and `url` are returned by `POST /shares` (the publish call),
NOT by this read endpoint — re-derive the URL from the path token if
needed (`https://okrapdf.com/playground/share/<token>`). Cached 60s. The
browser-facing URL is the same token on the `/playground/share/` path.

## Built-in facet variant IDs

The provider catalog has expanded substantially — as of 2026-04 the live
list (verified via `GET /plugins`) includes 18 variants: `textlayer`,
`llamaparse-fast`, `llamaparse-premium`, `llamaparse-agentic`,
`unstructured`, `google-document-ai-ocr`, `mistral-ocr`,
`mistral-ocr-annotated`, `reducto-parse`, `reducto-agentic`, `mineru`,
`gemini-3-flash-minimal`, `gemini-3-flash-high`, `gemini-3.1-pro`,
`gemini-3.1-flash-lite`, `azure-di-layout`, `chandra-ocr`,
`chandra-ocr-raw`.

**Always confirm via `GET /plugins?docId=<doc>` for the current session
+ doc** — the catalog rotates as new vendors are added, and gated
providers don't surface when unconfigured. Don't hardcode this list;
read it at runtime.

## Limits & gotchas

- **Per-session cost cap.** Two back-to-back compare runs can trip
  `"Playground cost limit reached. Reset the session to continue."` →
  `DELETE /session` and restart.
- **One uploaded doc per session.** `PLAYGROUND_MAX_UPLOAD_DOCS=1`.
  Reset to upload another.
- **Session TTL 1h.** Rehydrate with `{ sessionId }` before expiry or
  start fresh.
- **Per-IP rate limiting** via Cloudflare rate limiter on `/session`,
  `/init`, `/upload`. 429 = back off.
- **Cookies across calls.** The whole flow hangs off `okra_playground`.
  Use `-c -b` with curl or a real cookie jar in code.
- **Unknown variant IDs silently fall back to defaults.** A typo
  produces a successful run of a *different* vendor. Validate against
  `GET /plugins` first.
- **`/compare` is GET, `/shares` is POST.** Easy to swap by accident.
- **Snapshot key = (session, doc, variant-set).** Single-variant compare
  is required for share lookup. (See step 5.)
- **Seed docs don't count against upload limit** and don't need any
  binding step — pass the docId directly.

## Runnable recipe (TypeScript)

The okra monorepo ships a reference runner that encodes this whole
flow as a JSON recipe:

```json
{
  "pdf": "./paper.pdf",
  "plugins": ["llamaparse-fast", "github:acme/my-parser@main"],
  "share": { "ttl": "7d" },
  "out": "./run.json"
}
```

```bash
npx tsx scripts/playground/run.ts recipe.json
# prints the share URL on stdout; full run saved to run.json
```

Source: `scripts/playground/{run,lib,session,upload,compare,plugins,share}.ts`
in `okrapdf/okra`. Good starting point for automating share creation in CI
or a batch job. Session is persisted to `~/.okra/playground-session.json`
so repeat runs don't re-bootstrap.

## When NOT to use this

- You want **authenticated**, long-lived, revocable sharing → use the
  authenticated `/v1/documents` API with collection publishing.
- You want to share a **chat conversation** → not supported; build your
  own reader on top of the document API.
- You want the share to **update** when the source doc changes → shares
  are immutable snapshots. Create a new one.
