---
name: accessibility-audit
description: >
  Rules for the AltLens server-side accessibility audit pipeline:
  PageSpeed Insights API response parsing, Lighthouse JSON field mapping,
  Cheerio DOM extraction for images/audio/video, CORS proxy usage, batch
  processing limits, and the report data schema. Read this before touching
  app/api/scrape/route.ts, app/api/pagespeed/route.ts, or lib/db/schema.ts.
---

# Accessibility Audit Pipeline Skill

> **When to use:** Read this skill before writing or modifying any Route Handler in `app/api/`, the Drizzle schema in `lib/db/`, or any component that consumes audit result data.

---

## 1. Pipeline Overview

The audit pipeline has two server-side stages before any ML inference runs:

```
User submits URL
      │
      ▼
[Stage 1] GET /api/pagespeed
  └─ Calls PageSpeed Insights API
  └─ Extracts: overall score, categories, failing audits
      │
      ▼
[Stage 2] POST /api/scrape
  └─ Fetches target page HTML via CORS proxy
  └─ Cheerio parses: images, audio, video, text blocks
  └─ Returns: structured asset list + readability scores
      │
      ▼
[Client] Sends assets to Web Workers for ML captioning / transcription
      │
      ▼
[Stage 3] POST /api/report
  └─ Persists final report to SQLite via Drizzle
```

All Stage 1 and 2 work is server-side. Never call the PageSpeed API or use Cheerio client-side.

---

## 2. PageSpeed Insights API

### Route: `GET /api/pagespeed?url=<encoded-url>`

```ts
// app/api/pagespeed/route.ts
export async function GET(request: Request) {
  const { searchParams } = new URL(request.url)
  const targetUrl = searchParams.get('url')

  if (!targetUrl) {
    return Response.json({ error: 'Missing url parameter' }, { status: 400 })
  }

  const apiKey = process.env.PAGESPEED_API_KEY
  const endpoint = new URL(
    'https://www.googleapis.com/pagespeedonline/v5/runPagespeed',
  )
  endpoint.searchParams.set('url', targetUrl)
  endpoint.searchParams.set('strategy', 'desktop')
  // Request only the accessibility category to reduce response size
  endpoint.searchParams.set('category', 'ACCESSIBILITY')
  if (apiKey) endpoint.searchParams.set('key', apiKey)

  const res = await fetch(endpoint.toString(), { next: { revalidate: 0 } })

  if (!res.ok) {
    return Response.json(
      { error: 'PageSpeed API error', status: res.status },
      { status: 502 },
    )
  }

  const data = await res.json() as PageSpeedResponse
  return Response.json(extractPageSpeedSummary(data))
}
```

### Lighthouse JSON Field Mapping

The PageSpeed v5 response is large (~500 KB). Extract only the fields AltLens uses:

```ts
type PageSpeedSummary = {
  accessibilityScore: number        // 0–1 (multiply by 100 for display)
  failingAudits: FailingAudit[]
  lighthouseVersion: string
  fetchTime: string
}

type FailingAudit = {
  id: string           // e.g. 'image-alt', 'video-caption', 'audio-caption'
  title: string
  description: string
  score: number | null // null for informational audits
  displayValue: string | null
  items: AuditItem[]   // Specific elements that failed
}

type AuditItem = {
  node?: {
    snippet: string    // HTML snippet of the failing element
    selector: string   // CSS selector
    nodeLabel: string
  }
  url?: string         // For resource-level audits
}

function extractPageSpeedSummary(data: PageSpeedResponse): PageSpeedSummary {
  const lhr = data.lighthouseResult
  const accessibility = lhr.categories.accessibility

  // Only include audits that are relevant to AltLens (score < 1 or score === null)
  const RELEVANT_AUDIT_IDS = [
    'image-alt',
    'input-image-alt',
    'video-caption',
    'audio-caption',
    'aria-label',
    'aria-labelledby',
    'aria-hidden-body',
    'button-name',
    'link-name',
    'color-contrast',
    'document-title',
    'html-lang-valid',
    'label',
    'meta-description',
  ]

  const failingAudits: FailingAudit[] = RELEVANT_AUDIT_IDS
    .filter((id) => {
      const audit = lhr.audits[id]
      return audit && (audit.score === null || audit.score < 1)
    })
    .map((id) => {
      const audit = lhr.audits[id]
      return {
        id,
        title: audit.title,
        description: audit.description,
        score: audit.score,
        displayValue: audit.displayValue ?? null,
        items: audit.details?.items ?? [],
      }
    })

  return {
    accessibilityScore: accessibility.score,
    failingAudits,
    lighthouseVersion: lhr.lighthouseVersion,
    fetchTime: lhr.fetchTime,
  }
}
```

### Key Lighthouse Audit IDs to Watch

| Audit ID | What AltLens addresses |
|---|---|
| `image-alt` | Missing alt text — fixed by Vision Worker captioning |
| `input-image-alt` | `<input type="image">` missing alt |
| `video-caption` | Missing `<track>` element — fixed by Whisper transcription |
| `audio-caption` | Missing `<track>` for `<audio>` — fixed by Whisper |
| `aria-label` | Missing ARIA labels on interactive elements |
| `color-contrast` | Text/background contrast failures |
| `label` | Form inputs without associated labels |

---

## 3. Cheerio HTML Scraping — CORS Proxy Route

### Route: `POST /api/scrape`

```ts
// app/api/scrape/route.ts
import * as cheerio from 'cheerio'

const MAX_ASSET_BYTES = 10 * 1024 * 1024 // 10 MB

export async function POST(request: Request) {
  const body = await request.json() as { url: string; assetUrl?: string }

  // Mode 1: Scrape a page for its asset inventory
  if (!body.assetUrl) {
    return scrapePage(body.url)
  }

  // Mode 2: Proxy a single asset (image, audio, video)
  return proxyAsset(body.assetUrl)
}
```

### Page Scraping — Cheerio DOM Extraction

```ts
async function scrapePage(url: string): Promise<Response> {
  let html: string
  try {
    const res = await fetch(url, {
      headers: { 'User-Agent': 'AltLens-Accessibility-Bot/1.0' },
      signal: AbortSignal.timeout(10_000), // 10s timeout
    })
    if (!res.ok) {
      return Response.json({ error: 'Failed to fetch page' }, { status: 502 })
    }
    html = await res.text()
  } catch {
    return Response.json({ error: 'Network error fetching page' }, { status: 502 })
  }

  const $ = cheerio.load(html)
  const baseUrl = new URL(url).origin

  // ── Images ──────────────────────────────────────────────────────────────
  const images: ImageAsset[] = []
  $('img').each((_, el) => {
    const src = $(el).attr('src')
    const alt = $(el).attr('alt')
    const role = $(el).attr('role')
    if (!src) return
    // Skip decorative images (role=presentation or empty alt)
    const isDecorative = role === 'presentation' || alt === ''
    images.push({
      src: resolveUrl(src, baseUrl),
      alt: alt ?? null,
      isDecorative,
      needsCaptioning: !isDecorative && (alt === null || alt === undefined),
    })
  })

  // ── Audio ────────────────────────────────────────────────────────────────
  const audioElements: AudioAsset[] = []
  $('audio').each((_, el) => {
    const src = $(el).attr('src') ?? $('source', el).first().attr('src')
    const hasTrack = $('track[kind="captions"], track[kind="subtitles"]', el).length > 0
    if (!src) return
    audioElements.push({
      src: resolveUrl(src, baseUrl),
      hasTrack,
      needsTranscription: !hasTrack,
    })
  })

  // ── Video ────────────────────────────────────────────────────────────────
  const videoElements: VideoAsset[] = []
  $('video').each((_, el) => {
    const src = $(el).attr('src') ?? $('source', el).first().attr('src')
    const hasTrack = $('track[kind="captions"], track[kind="subtitles"]', el).length > 0
    if (!src) return
    videoElements.push({
      src: resolveUrl(src, baseUrl),
      hasTrack,
      needsTranscription: !hasTrack,
    })
  })

  // ── Text blocks for readability ───────────────────────────────────────────
  const textBlocks: TextBlock[] = []
  $('p, li, td, th, blockquote, figcaption').each((_, el) => {
    const text = $(el).text().trim()
    // Only score blocks with enough text to be meaningful
    if (text.split(/\s+/).length >= 15) {
      textBlocks.push({ text, selector: el.tagName })
    }
  })

  const result: ScrapeResult = {
    url,
    images,
    audioElements,
    videoElements,
    textBlocks,
    // Cap images at 50 for display; the client batches processing
    totalImageCount: images.length,
    imagesCapped: images.length > 50,
    images: images.slice(0, 50),
  }

  return Response.json(result)
}
```

### CORS Asset Proxy

```ts
async function proxyAsset(assetUrl: string): Promise<Response> {
  let res: globalThis.Response
  try {
    res = await fetch(assetUrl, {
      signal: AbortSignal.timeout(15_000),
    })
  } catch {
    return Response.json(
      { status: 'skipped', reason: 'cors-blocked' },
      { status: 200 }, // 200 so the client can record the skip without error handling
    )
  }

  // Check Content-Length before downloading
  const contentLength = Number(res.headers.get('content-length') ?? 0)
  if (contentLength > MAX_ASSET_BYTES) {
    return Response.json(
      { status: 'skipped', reason: 'too-large' },
      { status: 413 },
    )
  }

  const buffer = await res.arrayBuffer()

  // Double-check size after download (Content-Length may be absent)
  if (buffer.byteLength > MAX_ASSET_BYTES) {
    return new Response(JSON.stringify({ status: 'skipped', reason: 'too-large' }), {
      status: 413,
    })
  }

  return new Response(buffer, {
    headers: {
      'Content-Type': res.headers.get('content-type') ?? 'application/octet-stream',
      'Cache-Control': 'public, max-age=300',
    },
  })
}
```

### URL Resolution Helper

```ts
function resolveUrl(src: string, baseUrl: string): string {
  if (src.startsWith('http://') || src.startsWith('https://')) return src
  if (src.startsWith('//')) return `https:${src}`
  if (src.startsWith('/')) return `${baseUrl}${src}`
  return `${baseUrl}/${src}`
}
```

---

## 4. Scrape Result Types

```ts
// These types are shared between the Route Handler and client components.
// Define them in lib/types/audit.ts and import from there.

export type ImageAsset = {
  src: string
  alt: string | null
  isDecorative: boolean
  needsCaptioning: boolean
}

export type AudioAsset = {
  src: string
  hasTrack: boolean
  needsTranscription: boolean
}

export type VideoAsset = {
  src: string
  hasTrack: boolean
  needsTranscription: boolean
}

export type TextBlock = {
  text: string
  selector: string
}

export type SkippedAsset = {
  src: string
  status: 'skipped'
  reason: 'cors-blocked' | 'too-large'
}

export type ScrapeResult = {
  url: string
  images: ImageAsset[]
  audioElements: AudioAsset[]
  videoElements: VideoAsset[]
  textBlocks: TextBlock[]
  totalImageCount: number
  imagesCapped: boolean
}
```

---

## 5. Report Persistence Schema (Drizzle + SQLite)

```ts
// lib/db/schema.ts
import { sqliteTable, text, integer, real } from 'drizzle-orm/sqlite-core'

export const reports = sqliteTable('reports', {
  id: text('id').primaryKey(),            // UUID generated server-side
  url: text('url').notNull(),
  createdAt: integer('created_at', { mode: 'timestamp' }).notNull(),

  // PageSpeed summary
  accessibilityScore: real('accessibility_score'),  // 0.0–1.0
  lighthouseVersion: text('lighthouse_version'),
  failingAudits: text('failing_audits', { mode: 'json' })
    .$type<FailingAudit[]>(),

  // Scrape summary
  totalImages: integer('total_images').notNull().default(0),
  imagesNeedingAlt: integer('images_needing_alt').notNull().default(0),
  audioNeedingCaptions: integer('audio_needing_captions').notNull().default(0),
  videoNeedingCaptions: integer('video_needing_captions').notNull().default(0),

  // ML results (populated after Worker inference)
  imageCaptions: text('image_captions', { mode: 'json' })
    .$type<Record<string, string>>(),       // { [imageUrl]: caption }
  arabicCaptions: text('arabic_captions', { mode: 'json' })
    .$type<Record<string, string>>(),       // { [imageUrl]: arabicCaption }
  transcriptions: text('transcriptions', { mode: 'json' })
    .$type<Record<string, string>>(),       // { [mediaUrl]: vttContent }
  skippedAssets: text('skipped_assets', { mode: 'json' })
    .$type<SkippedAsset[]>(),

  // Readability
  complexTextCount: integer('complex_text_count').notNull().default(0),

  // Status
  status: text('status', { enum: ['pending', 'complete', 'error'] })
    .notNull()
    .default('pending'),
  errorMessage: text('error_message'),
})
```

---

## 6. Streaming Analysis — Route Handler Pattern

The `/api/scrape` scraping and PageSpeed calls happen fast enough to return in a single response. However, the overall analysis pipeline (scrape → batch caption → translate → persist) is long-running.

Stream progress events back to the client from a dedicated streaming endpoint:

```ts
// app/api/analyze/route.ts — streaming analysis coordinator
export async function POST(request: Request) {
  const encoder = new TextEncoder()

  const stream = new ReadableStream({
    async start(controller) {
      function emit(event: AnalysisEvent) {
        // Newline-delimited JSON (NDJSON) — easy to parse on the client
        controller.enqueue(encoder.encode(JSON.stringify(event) + '\n'))
      }

      try {
        emit({ type: 'STAGE', stage: 'pagespeed', message: 'Running PageSpeed audit...' })
        const pagespeed = await runPagespeedAudit(targetUrl)
        emit({ type: 'PAGESPEED_DONE', data: pagespeed })

        emit({ type: 'STAGE', stage: 'scrape', message: 'Scraping page structure...' })
        const scrape = await scrapePage(targetUrl)
        emit({ type: 'SCRAPE_DONE', data: scrape })

        emit({ type: 'STAGE', stage: 'report_created', reportId })

        controller.close()
      } catch (err) {
        emit({ type: 'ERROR', message: err instanceof Error ? err.message : String(err) })
        controller.close()
      }
    },
  })

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/plain; charset=utf-8',
      'X-Content-Type-Options': 'nosniff',
      // Disable Nginx/CDN buffering so chunks arrive immediately
      'X-Accel-Buffering': 'no',
    },
  })
}
```

### Client-side stream reading
```ts
// In a 'use client' component
const response = await fetch('/api/analyze', { method: 'POST', body })
const reader = response.body!.getReader()
const decoder = new TextDecoder()

while (true) {
  const { done, value } = await reader.read()
  if (done) break
  const lines = decoder.decode(value).split('\n').filter(Boolean)
  for (const line of lines) {
    const event = JSON.parse(line) as AnalysisEvent
    dispatch(event) // Update React state / ARIA live region
  }
}
```

---

## 7. 50-Image Batch Cap — Processing Rules

- Cheerio extraction caps at **50 images** per page (configurable via `MODELS.vision.maxImages`).
- If `totalImageCount > 50`, the UI must display a warning: *"Showing first 50 of N images."*
- Images are processed in batches of **4** concurrently (configurable via `MODELS.vision.batchSize`).
- Each batch is dispatched as individual `CAPTION_IMAGE` Worker requests — the batch logic lives in the React component, not the Worker.
- Skipped assets (CORS-blocked or >10 MB) are still recorded in the report with their `reason`.

---

## 8. Common Pitfalls

| Pitfall | Correct approach |
|---|---|
| Using `eval` or regex to parse HTML | Always use Cheerio — it handles malformed HTML safely |
| Hardcoding `http://` when resolving relative URLs | Use the `resolveUrl()` helper — some assets use `//` protocol-relative URLs |
| Treating empty `alt=""` as missing alt text | `alt=""` is intentional decoration — skip captioning for these |
| Forgetting to handle `<source>` inside `<audio>`/`<video>` | `<audio src>` may be absent; always check `<source>` children |
| Not setting `X-Accel-Buffering: no` on streaming responses | Nginx/reverse proxies buffer by default, breaking the stream UX |
| Reading `PAGESPEED_API_KEY` client-side | This key must only be read inside the Route Handler — never in a Client Component or hook |
| Storing large HTML or full Lighthouse JSON in SQLite | Only persist the `extractPageSpeedSummary` result — not the raw API response |
