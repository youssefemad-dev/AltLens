---
name: transformers-js
description: >
  Guidance for integrating @xenova/transformers into AltLens Web Workers.
  Covers: verified ONNX model IDs, singleton pipeline pattern, WebGPU/WASM
  backend detection, Cache API caching behaviour, progress_callback streaming,
  and batch processing constraints. Read this before touching any worker file.
---

# Transformers.js Integration Skill

> **When to use:** Read this skill before writing or modifying any file in `workers/`. Also read `lib/models.config.ts` and `lib/worker-protocol.ts` first.

---

## 1. Canonical Model Registry

All model IDs are defined in `lib/models.config.ts`. The table below is the ground truth — do **not** invent or guess model IDs.

| Task | Model ID | Approx. size | Format |
|---|---|---|---|
| Image captioning (vision) | `Xenova/vit-gpt2-image-captioning` | ~80 MB | ONNX |
| Speech recognition (ASR) | `Xenova/whisper-tiny` | ~150 MB | ONNX |
| EN→AR translation | `Xenova/opus-mt-en-ar` | ~300 MB | ONNX |

These identifiers are verified ONNX-exported checkpoints hosted on the Hugging Face Hub under the `Xenova` namespace. Do not substitute alternative model IDs without an explicit architectural decision and update to `models.config.ts`.

### `lib/models.config.ts` shape
```ts
export const MODELS = {
  vision: {
    id: 'Xenova/vit-gpt2-image-captioning',
    task: 'image-to-text',
  },
  whisper: {
    id: 'Xenova/whisper-tiny',
    task: 'automatic-speech-recognition',
  },
  translation: {
    id: 'Xenova/opus-mt-en-ar',
    task: 'translation',
  },
} as const satisfies Record<string, { id: string; task: string }>

export type ModelKey = keyof typeof MODELS
```

---

## 2. Worker File Structure

Every worker file must follow this skeleton exactly:

```ts
// workers/vision.worker.ts
import * as transformers from '@xenova/transformers'
import type { WorkerRequest, WorkerResponse } from '../lib/worker-protocol'
import { MODELS } from '../lib/models.config'

// --- Singleton pipeline ---
let _pipeline: Awaited<ReturnType<typeof transformers.pipeline>> | null = null

async function getPipeline(
  onProgress: (event: transformers.ProgressInfo) => void,
): Promise<Awaited<ReturnType<typeof transformers.pipeline>>> {
  if (!_pipeline) {
    _pipeline = await transformers.pipeline(
      MODELS.vision.task,
      MODELS.vision.id,
      { progress_callback: onProgress },
    )
  }
  return _pipeline
}

// --- Message handler ---
self.addEventListener('message', async (event: MessageEvent<WorkerRequest>) => {
  const req = event.data

  // Only handle requests addressed to this worker
  if (req.type !== 'CAPTION_IMAGE') return

  // Progress forwarder — posts download/load events back to the main thread
  const onProgress = (info: transformers.ProgressInfo) => {
    const response: WorkerResponse = {
      id: req.id,
      type: 'MODEL_PROGRESS',
      payload: info,
    }
    self.postMessage(response)
  }

  try {
    const pipe = await getPipeline(onProgress)
    const result = await pipe(req.payload.imageUrl)
    const response: WorkerResponse = {
      id: req.id,
      type: 'CAPTION_RESULT',
      payload: { caption: (result as Array<{ generated_text: string }>)[0].generated_text },
    }
    self.postMessage(response)
  } catch (err) {
    const response: WorkerResponse = {
      id: req.id,
      type: 'ERROR',
      payload: { message: err instanceof Error ? err.message : String(err) },
    }
    self.postMessage(response)
  }
})
```

Apply the same pattern for `whisper.worker.ts` and `translation.worker.ts`, substituting the model key and result shape accordingly.

---

## 3. WebGPU / WASM Backend Detection

Detect and configure the inference backend **once at Worker startup** before the first `pipeline()` call:

```ts
// At the top of each worker file, before getPipeline is called
async function configureBackend(): Promise<'webgpu' | 'wasm'> {
  if ('gpu' in navigator) {
    try {
      const adapter = await (navigator as Navigator & { gpu: GPU }).gpu.requestAdapter()
      if (adapter) {
        // WebGPU is available — disable WASM proxy so ONNX uses the GPU
        transformers.env.backends.onnx.wasm.proxy = false
        return 'webgpu'
      }
    } catch {
      // GPU request failed — fall through to WASM
    }
  }
  // WASM CPU fallback
  transformers.env.backends.onnx.wasm.proxy = true
  return 'wasm'
}
```

Post the detected backend back to the main thread on first startup:
```ts
const backend = await configureBackend()
const initResponse: WorkerResponse = {
  id: 'init',
  type: 'BACKEND_DETECTED',
  payload: { backend },
}
self.postMessage(initResponse)
```

The Model Sidebar component reads this and displays `WebGPU` or `CPU (WASM)` as a badge next to each model.

---

## 4. Model Caching — Cache API Behaviour

Transformers.js caches model shards using the browser **Cache API** (accessible via `caches.open()`), NOT IndexedDB. Key points:

- Cache keys follow the pattern: `hf://Xenova/<model-id>/<filename>`
- Caching is **automatic** — do not implement custom caching logic.
- Model files persist across page refreshes and browser restarts (until the user clears site data).
- The Cache API is available inside Web Workers — no extra setup required.

### Estimating cache size (for the sidebar UI)
```ts
// Run in the main thread, not in the worker
async function estimateCacheSize(): Promise<number> {
  const cache = await caches.open('transformers-cache')
  const keys = await cache.keys()
  let totalBytes = 0
  for (const req of keys) {
    const resp = await cache.match(req)
    if (resp) {
      const buf = await resp.arrayBuffer()
      totalBytes += buf.byteLength
    }
  }
  return totalBytes // bytes
}
```

Surface this as human-readable MB in the sidebar (e.g., `"~530 MB cached"`).

> **Warning:** Total weight for all three models is approximately 530 MB. Warn the user before triggering downloads if their connection appears slow (check `navigator.connection.effectiveType` if available).

---

## 5. `progress_callback` — Streaming Download Progress

The `progress_callback` receives a `ProgressInfo` object from `@xenova/transformers`. Its shape varies by event:

```ts
// Downloading a shard
{ status: 'downloading', name: string, file: string, loaded: number, total: number, progress: number }

// Shard download complete
{ status: 'done', name: string, file: string }

// Model loaded into memory
{ status: 'ready', task: string, model: string }

// Initialising (pre-download)
{ status: 'initiate', name: string, file: string }
```

The `progress` field (when present) is a number from `0` to `100`.

Forward each callback invocation as a `MODEL_PROGRESS` WorkerResponse — the main thread accumulates these to drive the sidebar progress bars. See `lib/worker-protocol.ts` for the full type.

---

## 6. Image Captioning — Batch Processing Rules

Images are processed in **concurrent batches** with a configurable batch size (default: `4`).

- Batch size is defined in `lib/models.config.ts` as `MODELS.vision.batchSize`.
- The main thread enqueues individual `CAPTION_IMAGE` requests sequentially — it does not send arrays.
- The Worker processes them one at a time (the pipeline is not parallelisable within a single Worker without multiple instances).
- The UI renders a live progress bar based on `completed / total` image count tracked in React state.
- If a single image fails, post an `ERROR` response for that request ID and continue processing the rest. Do not abort the batch.

---

## 7. Whisper ASR — Audio Input Constraints

- Whisper-tiny expects audio as a `Float32Array` sampled at **16 kHz mono**.
- The main thread is responsible for decoding the audio (via `AudioContext.decodeAudioData`) before sending it to the Worker — do not attempt to decode audio inside the Worker.
- The decoded `Float32Array` is transferable — use `postMessage(msg, [buffer])` to transfer the underlying `ArrayBuffer` without copying it (zero-copy transfer).
- Output: Whisper returns a `text` string. The Worker must convert this to `.vtt` format before posting the result.

### VTT conversion
```ts
// Minimal VTT formatter for Whisper output
function toVTT(text: string): string {
  // Whisper-tiny does not produce timestamps at this model size.
  // Wrap the full transcript in a single cue.
  return `WEBVTT\n\n00:00:00.000 --> 99:59:59.999\n${text.trim()}\n`
}
```

---

## 8. Translation — Text Chunking

- `opus-mt-en-ar` has a max sequence length of **512 tokens**.
- If the input alt text is likely to exceed this (rough heuristic: >400 characters), split it at sentence boundaries before translating and concatenate results.
- The Worker handles chunking internally — the main thread sends a single string per request.

---

## 9. Common Pitfalls

| Pitfall | Correct approach |
|---|---|
| Calling `pipeline()` on every message | Use the module-level singleton — instantiate once, reuse always |
| Not forwarding `progress_callback` | Always pass it so the sidebar can show download progress |
| Importing workers in Server Components | Workers are browser-only — instantiate in Client Components with `new Worker(new URL(...), { type: 'module' })` |
| Transferring `ImageData` or `Float32Array` without transfer list | Use `postMessage(msg, [buffer])` to avoid copying large buffers |
| Swallowing errors silently | Always catch and post a typed `ERROR` response back to the main thread |
| Using `Blob` URLs for audio without cleanup | Always call `URL.revokeObjectURL()` after the Worker has received the data |
