---
name: okra-create
description: Generate designed PDFs via OkraPDF's pdf-render-agent — baked-design reports (magazine, editorial, report, poster), custom agent-written HTML covers via Cloudflare Browser Rendering, or fully composed output through codemode. Triggered by "create a pdf", "generate a report", "design a cover", "make a pdf with html", or any PDF-generation task where visual quality matters.
---

# OkraPDF — Create

One Worker, three modes to produce designed PDFs:

1. **`POST /render`** — baked design system from [minimax-pdf]; pick a `type` (report, magazine, editorial, poster…) and send content blocks. Container renders body in Python (reportlab), Browser Rendering renders the cover, Worker merges.
2. **`POST /browser/html-to-pdf`** — agent writes raw HTML; Chromium returns a PDF. Pixel-perfect custom covers and one-pagers.
3. **`POST /codemode/run`** — agent submits JS that composes `pdf.render` + `browser.htmlToPdf` + `pdf.merge` in one request. Custom cover + baked body in a single call.

[minimax-pdf]: https://skills.sh/minimax-ai/skills

## Architecture

```
Request → Worker
           │
           ├─ DO Container (Python: palette → cover-html → body → merge)
           └─ env.BROWSER (Chromium via @cloudflare/puppeteer)
```

- **Python container (~520MB image)** — reportlab + matplotlib + pypdf.
  Handles `/tokens`, `/cover-html`, `/body`, `/merge` endpoints.
  Durable-Object-backed, warm after first request, sleeps after 5 min.
- **Cloudflare Browser Rendering** — renders agent-authored HTML via
  `puppeteer.launch(env.BROWSER, { keep_alive: 600000 })`.
- **Worker Loader (codemode)** — sandboxed JS runs with injected host fns.

## Mode 1: baked designs (`POST /render`)

One call in, one merged PDF out. Worker orchestrates all stages.

```bash
curl -X POST https://pdf-render-agent.steventsao.workers.dev/render \
  -H 'content-type: application/json' \
  --data @spec.json -o out.pdf
```

**Request shape:**

```jsonc
{
  "title": "Q3 Review",
  "type": "report",        // or: proposal, resume, portfolio, academic,
                           //     general, minimal, stripe, diagonal,
                           //     frame, editorial, magazine, darkroom,
                           //     terminal, poster
  "author": "Strategy Team",
  "date": "April 2026",
  "accent": "#2D5F8A",     // muted hex; avoid default blue
  "cover_bg": "#1A1A1A",   // optional
  "abstract": "...",       // magazine/darkroom only
  "cover_image": "https://...jpg",  // magazine/darkroom/poster
  "skipCover": false,      // true = body-only (~400ms warm)
  "content": [             // minimax-pdf block types:
    { "type": "h1", "text": "Executive Summary" },
    { "type": "body", "text": "Justified paragraph with <b>markup</b>." },
    { "type": "callout", "text": "Highlighted insight." },
    { "type": "bullet", "text": "Point one." },
    { "type": "numbered", "text": "Counter resets automatically." },
    { "type": "table",
      "headers": ["Col", "B", "C"],
      "col_widths": [0.5, 0.25, 0.25],  // fractions of usable width
      "rows": [["a","1","2"]],
      "caption": "..." },
    { "type": "chart", "chart_type": "bar",
      "labels": ["Q1","Q2"], "datasets": [{"label":"Rev","values":[100,200]}] },
    { "type": "code", "text": "print('x')", "language": "python" },
    { "type": "math", "text": "E = mc^2", "label": "(1)" },
    { "type": "divider" }, { "type": "pagebreak" }, { "type": "spacer", "pt": 18 }
  ]
}
```

**Perf (warm container):**
- `skipCover: true` → **~400ms-2s** (body only, via container alone)
- full pipeline → **~6-8s** (adds Browser Rendering cover + merge)

## Mode 2: custom HTML cover (`POST /browser/html-to-pdf`)

Agent writes the HTML. Browser Rendering returns a PDF.

```bash
curl -X POST https://pdf-render-agent.steventsao.workers.dev/browser/html-to-pdf \
  -H 'content-type: application/json' \
  -d '{"html":"<!doctype html>...","width":794,"height":1123}' \
  -o cover.pdf
```

**Request:**
```jsonc
{
  "html": "<!doctype html>...",
  "width": 794,              // px, default 794 (A4 @ 96dpi)
  "height": 1123,            // px, default 1123
  "waitForSelector": "#ready", // optional
  "waitForTimeoutMs": 5000
}
```

**Non-negotiable template rules** (see `templates/editorial-hero.html`):

1. `@page { size: 794px 1123px; margin: 0; }` — locks PDF page dims.
2. `html, body { width: 794px; height: 1123px; overflow: hidden; }` — viewport anchor.
3. Use **CSS Grid rows** (no absolute positioning) to guarantee no overlaps.
4. `grid-template-columns: minmax(0, 1fr) 220px;` — the `minmax(0, …)`
   prevents content from expanding a flexible column past its share.
5. `overflow: hidden` on the page container — last-line defense against bleeds.
6. `@import` Google Fonts at the top; Worker uses `waitUntil: "domcontentloaded"`
   (not `networkidle0` — fonts can stall that).

See `templates/editorial-hero.html` for a working magazine-cover reference.

## Mode 3: agent composition via codemode (`POST /codemode/run`)

Agent submits JS. Worker injects host fns: `pdf.render`, `pdf.merge`,
`browser.htmlToPdf`. Sandbox has `globalOutbound: null` — only the host fns
are reachable.

```js
// user.js (the string you send as body.code)
export default async function ({ pdf, browser }) {
  const cover = await browser.htmlToPdf({ html: MY_COVER_HTML });
  const body  = await pdf.render({ ...spec, skipCover: true });
  const out   = await pdf.merge([cover.pdfBase64, body.pdfBase64]);
  return { pdfBase64: out.pdfBase64, size: out.size };
}
```

```bash
curl -X POST https://pdf-render-agent.steventsao.workers.dev/codemode/run \
  -H 'content-type: application/json' \
  -d '{"code":"...","returnPdf":true}' -o final.pdf
```

Build the request from Python/Node with `JSON.stringify` to embed HTML
and spec safely:

```python
req = {
  "code": f"""
    const COVER_HTML = {json.dumps(cover_html)};
    const BODY_SPEC  = {json.dumps(body_spec)};
    export default async ({{ pdf, browser }}) => {{
      const cover = await browser.htmlToPdf({{ html: COVER_HTML }});
      const body  = await pdf.render(BODY_SPEC);
      const out   = await pdf.merge([cover.pdfBase64, body.pdfBase64]);
      return {{ pdfBase64: out.pdfBase64, size: out.size }};
    }};
  """,
  "returnPdf": True
}
```

## Accent color — pick from content, not a palette

The minimax-pdf design system rule: **don't default to blue**. Muted,
desaturated, pulled from the document's industry.

| Context | Suggested hex |
|---|---|
| Legal / compliance / finance | `#1C3A5E`, `#2E3440` |
| Healthcare | `#2A6B5A`, `#3A7D6A` |
| Tech / engineering | `#2D5F8A`, `#3D4F8A` |
| Sustainability | `#2E5E3A`, `#4A5E2A` |
| Creative / arts | `#6B2A35`, `#5A2A6B`, `#8A3A2A` |
| Academic / research | `#2A5A6B`, `#2A4A6B` |
| Premium / luxury | `#1A1208`, `#4A3820` |

## Gotchas learned the hard way

**Browser Rendering hangs on first call.**
- Cause: missing `keep_alive` on `puppeteer.launch()` or `waitUntil: "networkidle0"` with `@import`ed fonts.
- Fix (already in Worker): `puppeteer.launch(env.BROWSER, { keep_alive: 600000 })` + `setContent(html, { waitUntil: "domcontentloaded" })`.

**Content clips on the right edge of the PDF.**
- Cause: Worker didn't set viewport before `setContent`; Chrome used its default viewport, layout math didn't match PDF size.
- Fix (in Worker): `page.setViewport({ width: 794, height: 1123 })` before `setContent`; `page.pdf({ width: "794px", height: "1123px" })`.

**Table cells wrap digits like `90.74` → `90.7 / 4`.**
- Cause: `col_widths` fraction too small for the numeric precision.
- Fix: give numeric columns ≥ 0.10; data fits at 0.11-0.12 for 4-char decimals.

**"specialized" wraps as "specializ/ed" in table rows.**
- Cause: Category col < 0.13 of usable width.
- Fix: either widen to 0.13+ or abbreviate to "spec" / "VLM".

**Reportlab justification stretches short paragraphs.**
- Every `body` block is full-justified. For closing 1-2 sentence paragraphs, fold into the previous body block or accept the rivering.

**Trailing caption block orphans to a nearly-empty last page.**
- Drop the trailing caption or inline it in the last body paragraph.

## Worker endpoints at a glance

```
GET  /health                 container liveness
GET  /browser/ping           launch+close diagnostic (~3.5s)
GET  /browser/limits         puppeteer.limits(env.BROWSER)

POST /render                 body: RenderSpec         -> application/pdf
POST /browser/html-to-pdf    body: {html, width?...}  -> application/pdf
POST /codemode/run           body: {code, returnPdf?} -> json | application/pdf
```

## Deploy your own Worker

The reference impl lives in `okra` meta-repo at
`apps/api/packages/pdf-render-agent/`. Wrangler config requires:

```jsonc
"containers": [{ "class_name": "PdfRender", "image": "./Dockerfile",
                 "max_instances": 5, "instance_type": "standard-1" }],
"durable_objects": { "bindings": [{ "name": "PDF_RENDER", "class_name": "PdfRender" }] },
"worker_loaders": [{ "binding": "LOADER" }],
"browser":        { "binding": "BROWSER" }
```

Needs: Workers Paid (Containers + Browser Rendering both require it).
First deploy pushes the ~520MB container image to CF's registry.

## Template library

- `templates/editorial-hero.html` — magazine cover with big display number,
  side attribution, title + lede + TOC + footer. CSS Grid rows, all safety rails in place.
  Works as-is via `POST /browser/html-to-pdf`. Swap the content between `<body>` tags
  or parametrize later with a builder function.
