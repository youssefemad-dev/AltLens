<!-- BEGIN:nextjs-agent-rules -->

## This is NOT the Next.js you know

This version has breaking changes — APIs, conventions, and file structure may all differ from your training data. Read the relevant guide in `node_modules/next/dist/docs/` (resolved from this file's directory; in monorepos the `next` package may not be visible from the repo root) before writing any code. Heed deprecation notices.

This block is written and re-added by `next dev` — verify at `node_modules/next/dist/server/lib/generate-agent-files.js`. Removing it from a diff only re-creates the uncommitted change; committing it with your work keeps the tree clean.

<!-- END:nextjs-agent-rules -->

---

# AltLens — Agentic System Rules

> **Scope:** These rules govern ALL code written in this repository. They are non-negotiable unless a rule explicitly marks an option as context-dependent. When in doubt, ask — do not assume.

---

## 1. Tech Stack — Canonical Versions

| Layer | Package | Version | Notes |
|---|---|---|---|
| Framework | `next` | **16.4.0** | App Router only — no Pages Router |
| UI runtime | `react` / `react-dom` | **19.3.0** | `use()`, `useOptimistic`, `useActionState` are stable |
| Language | TypeScript | **^5** | `strict: true` is mandatory — see §3 |
| Styling | `tailwindcss` | **^4** | Tailwind v4 syntax (CSS-first config in `app/globals.css`) |
| Bundler (dev) | Turbopack | built-in | `next dev --turbo` is the default dev command |
| ORM | Drizzle + `better-sqlite3` | to be installed | SQLite for report persistence |
| HTML parser | `cheerio` | to be installed | server-side only, never import in Client Components |
| ML runtime | `@xenova/transformers` | to be installed | Web Worker only — see §5 |

### Tailwind v4 note
Tailwind v4 uses a CSS-first config — there is **no `tailwind.config.js`**. All theme customisation lives in `app/globals.css` via `@theme { … }` blocks. The `@tailwindcss/turbopack` dev dependency is already installed and wires Tailwind into Turbopack automatically. Do not create `tailwind.config.js` or `tailwind.config.ts`.

---

## 2. Repository Structure

```
altlens/
├── app/
│   ├── layout.tsx               # Root layout — no 'use client'
│   ├── page.tsx                 # Landing / URL input  (route: /)
│   ├── globals.css              # Tailwind v4 CSS-first config + global styles
│   ├── analyze/
│   │   └── page.tsx             # Analysis in-progress view  (route: /analyze)
│   ├── report/
│   │   └── [id]/
│   │       └── page.tsx         # Persisted report view  (route: /report/[id])
│   └── api/
│       ├── scrape/route.ts      # POST — Cheerio HTML scrape + CORS proxy
│       ├── pagespeed/route.ts   # GET  — PageSpeed Insights API proxy
│       └── report/
│           ├── route.ts         # POST — create report in SQLite
│           └── [id]/route.ts    # GET  — fetch report by id
├── lib/
│   ├── models.config.ts         # Single source of truth for all HF model IDs
│   ├── worker-protocol.ts       # WorkerRequest / WorkerResponse discriminated unions
│   ├── readability.ts           # Flesch-Kincaid heuristic — pure function, no deps
│   └── db/
│       ├── schema.ts            # Drizzle schema
│       └── client.ts            # Drizzle SQLite client singleton
├── workers/
│   ├── vision.worker.ts         # ViT-GPT2 image captioning
│   ├── whisper.worker.ts        # Whisper ASR → .vtt subtitles
│   └── translation.worker.ts   # opus-mt-en-ar English→Arabic
├── components/
│   ├── ui/                      # Reusable primitives (Button, Badge, etc.)
│   ├── model-sidebar/           # Persistent model status panel
│   └── analysis/                # Domain-specific analysis components
└── .agents/
    └── skills/
        ├── transformers-js/SKILL.md
        ├── web-worker-bridge/SKILL.md
        └── accessibility-audit/SKILL.md
```

---

## 3. TypeScript Rules

- **`strict: true` is mandatory** in `tsconfig.json`. Every file must compile cleanly.
- **No `any`** — use `unknown` and narrow explicitly, or define a proper type.
- **No non-null assertions (`!`)** unless accompanied by an inline comment explaining why the value is guaranteed non-null.
- **Discriminated unions are required** for all Worker message passing. See `lib/worker-protocol.ts` and the `web-worker-bridge` skill.
- Use `satisfies` and `as const` liberally for config objects (e.g., `models.config.ts`).
- Page `params` are `Promise<{ … }>` in Next.js 16 — always `await params` before reading fields, or pass the promise down and resolve it inside a `<Suspense>` boundary.

---

## 4. Next.js 16 App Router — Non-Negotiable Rules

### Server vs Client Components
- **Default is Server Component.** Only add `'use client'` when a component uses browser APIs, React state/effects, or event handlers.
- **Never import `cheerio`, `better-sqlite3`, or Node.js built-ins** from a Client Component or Web Worker. These are server-only.
- **Never import `@xenova/transformers`** from a Server Component or Route Handler. It runs in browser Web Workers only.

### Route Handlers (`app/api/**/route.ts`)
- All Route Handlers use the Web fetch-API signature: `export async function GET(request: Request)`, `POST`, etc.
- **PageSpeed API key** (`PAGESPEED_API_KEY`) must only be read inside Route Handlers — never exposed to the client bundle.
- Streaming Route Handlers use `new ReadableStream({ async start(controller) { … } })` returned as `new Response(stream, { headers })`. Set `'Content-Type': 'text/plain; charset=utf-8'` and `'X-Content-Type-Options': 'nosniff'`.
- The CORS proxy Route Handler (`/api/scrape`) must enforce a **10 MB max asset size** guard. Reject requests with `{ error: 'Asset too large' }` (HTTP 413) before proxying.
- All Route Handlers must return proper HTTP status codes. Do **not** return `200 OK` for error states.

### Params are Promises
```tsx
// Correct in Next.js 16
export default async function ReportPage({
  params,
}: {
  params: Promise<{ id: string }>
}) {
  const { id } = await params
  // ...
}
```

### Caching
- Use `'use cache'` directive with `cacheTag` / `cacheLife` for report reads (stable, id-keyed data).
- Use `updateTag` inside Server Functions after mutations (not `revalidatePath` unless necessary).
- Dynamic reads (`cookies()`, `headers()`, request-time `searchParams`) must be pushed down to the component that needs them and wrapped in `<Suspense>`.

---

## 5. ML Inference — Zero Main-Thread Policy

> **This is the most critical rule in the entire codebase. Violation will freeze the UI.**

### The Rule
**ALL `@xenova/transformers` pipeline calls MUST run inside a Web Worker.**
- No exceptions. No `pipeline()` calls in React components, hooks, Server Components, or Route Handlers.
- The main React thread communicates with Workers exclusively via `postMessage` using the typed protocol defined in `lib/worker-protocol.ts`.

### Worker Topology
Three dedicated Workers, one per model domain:

| Worker file | Model | Task |
|---|---|---|
| `workers/vision.worker.ts` | `Xenova/vit-gpt2-image-captioning` | Generate alt text for images |
| `workers/whisper.worker.ts` | `Xenova/whisper-tiny` | Transcribe audio/video to .vtt |
| `workers/translation.worker.ts` | `Xenova/opus-mt-en-ar` | Translate alt text EN to AR |

### Lazy Instantiation
Workers must be **lazily created** — do not instantiate all three on page load. Create each Worker the first time the user triggers a task that needs it.

### Inference Backend Priority
1. **WebGPU** (detect with `'gpu' in navigator`): set `env.backends.onnx.wasm.proxy = false` and use WebGPU device.
2. **WASM CPU fallback**: if WebGPU is unavailable, Transformers.js falls back automatically when `env.backends.onnx.wasm.proxy = true`.
3. Surface the active backend in the Model Sidebar UI as a badge: `WebGPU` or `CPU (WASM)`.

### Singleton Pattern (inside Workers)
Each Worker must use a module-level singleton for its pipeline to avoid reloading model weights on every message:
```ts
// Inside a worker file — correct pattern
let _pipeline: unknown = null

async function getPipeline() {
  if (!_pipeline) {
    _pipeline = await transformers.pipeline(/* ... */)
  }
  return _pipeline
}
```

### Model Caching
Transformers.js uses the browser's **Cache API** (not IndexedDB) for model weights. Keys follow the pattern `hf://Xenova/<model-id>/...`. This is automatic — do not implement custom caching. Do surface estimated cache usage in the Model Sidebar.

---

## 6. Accessibility — AltLens Must Score 100/100

AltLens is an accessibility tool. It must itself be fully accessible.

### Mandatory Requirements
- **Lighthouse Accessibility score: 100/100** on all three routes (`/`, `/analyze`, `/report/[id]`).
- **Keyboard navigation**: every interactive element must be reachable and operable via keyboard alone. Tab order must be logical.
- **ARIA live regions**: all asynchronous status updates (model download progress, analysis steps, streaming results) must be announced via `aria-live="polite"` regions or `aria-live="assertive"` for errors.
- **Colour contrast**: minimum WCAG AA (4.5:1 for normal text, 3:1 for large text). Verify with Lighthouse.
- **Focus management**: after navigation or modal open/close, focus must be programmatically moved to the appropriate element.
- **`alt` text on all images** — including dynamically generated captioning results shown in the UI.
- **Semantic HTML**: use `<main>`, `<nav>`, `<section>`, `<article>`, `<header>`, `<footer>` appropriately. One `<h1>` per page.
- **No `tabindex` values greater than 0.**
- **Form labels**: every `<input>`, `<select>`, and `<textarea>` must have an associated `<label>` (or `aria-label` / `aria-labelledby`).

### Model Sidebar Accessibility
The persistent model status sidebar must have:
- `role="complementary"` and `aria-label="AI Model Status"`
- Each model status item: `role="status"` or wrapped in an `aria-live="polite"` region
- Download progress bars: `role="progressbar"` with `aria-valuenow`, `aria-valuemin`, `aria-valuemax`, and `aria-label`

---

## 7. Readability Feature

Readability scoring is implemented as a **pure heuristic** — no ML model is loaded for this feature.

- Implementation lives in `lib/readability.ts` as a synchronous exported function.
- Algorithm: Flesch-Kincaid Reading Ease score (0–100 scale).
- Threshold: flag text blocks scoring below **50** (college-level or harder) as "Complex Text".
- The function runs on the main thread after Cheerio extracts text nodes server-side.
- Do **not** add a transformer classifier for readability without an explicit architectural decision.

---

## 8. Error Handling

- **Never silence errors with empty `catch` blocks.**
- Worker errors must be posted back to the main thread as a typed `WorkerResponse` with `type: 'ERROR'` — they must never go unhandled.
- CORS-blocked or oversized assets caught by the proxy Route Handler must be recorded in the analysis report as `{ status: 'skipped', reason: 'cors-blocked' | 'too-large' }` — never silently dropped.
- Use `notFound()` **before** any `<Suspense>` boundary in dynamic route pages to get a real HTTP 404 (not a streamed 200 with injected error).

---

## 9. Environment Variables

| Variable | Where used | Required |
|---|---|---|
| `PAGESPEED_API_KEY` | Route Handler `/api/pagespeed` only | Yes (in production) |
| `DATABASE_URL` | `lib/db/client.ts` only | Defaults to `./altlens.db` |

- Never read `process.env` inside Client Components or Web Workers.
- All env vars are accessed server-side only.
- Document all required env vars in `.env.example`.

---

## 10. Skill Files

Before writing code for the following domains, read the corresponding skill file:

| Domain | Skill file |
|---|---|
| `@xenova/transformers` integration | `.agents/skills/transformers-js/SKILL.md` |
| Worker to React message passing | `.agents/skills/web-worker-bridge/SKILL.md` |
| PageSpeed / Cheerio audit pipeline | `.agents/skills/accessibility-audit/SKILL.md` |
