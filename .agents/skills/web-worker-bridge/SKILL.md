---
name: web-worker-bridge
description: >
  Typed message-passing standards between React Client Components and the three
  AltLens Web Workers (vision, whisper, translation). Covers: WorkerRequest and
  WorkerResponse discriminated union types, the useWorker React hook pattern,
  request ID correlation, event contracts for model download progress, and
  zero-copy buffer transfer rules. Read this before touching lib/worker-protocol.ts
  or any component that communicates with a worker.
---

# Web Worker Bridge Skill

> **When to use:** Read this skill before writing or modifying `lib/worker-protocol.ts`, the `useWorker` hook, or any React component that sends messages to or receives messages from a Web Worker.

---

## 1. Core Types — `lib/worker-protocol.ts`

This file is the **single source of truth** for all Worker ↔ main-thread communication. Both Worker files and React hooks import from here. Never define message types inline anywhere else.

```ts
// lib/worker-protocol.ts

// ─── Inbound (Main → Worker) ────────────────────────────────────────────────

export type CaptionImageRequest = {
  id: string           // Unique request ID (crypto.randomUUID())
  type: 'CAPTION_IMAGE'
  payload: {
    imageUrl: string   // Proxied URL via /api/scrape, or data URL
  }
}

export type TranscribeRequest = {
  id: string
  type: 'TRANSCRIBE'
  payload: {
    audioBuffer: Float32Array  // 16 kHz mono PCM — transferable
    mimeType: string
  }
}

export type TranslateRequest = {
  id: string
  type: 'TRANSLATE'
  payload: {
    text: string       // English alt text to translate
  }
}

/** Union of all valid messages sent FROM the main thread TO a Worker */
export type WorkerRequest =
  | CaptionImageRequest
  | TranscribeRequest
  | TranslateRequest

// ─── Outbound (Worker → Main) ───────────────────────────────────────────────

export type ModelProgressResponse = {
  id: string           // Matches the originating request ID, or 'init' for startup
  type: 'MODEL_PROGRESS'
  payload: {
    status: 'initiate' | 'downloading' | 'done' | 'ready'
    file?: string
    progress?: number  // 0–100
    loaded?: number    // bytes
    total?: number     // bytes
  }
}

export type BackendDetectedResponse = {
  id: 'init'
  type: 'BACKEND_DETECTED'
  payload: {
    backend: 'webgpu' | 'wasm'
  }
}

export type CaptionResultResponse = {
  id: string
  type: 'CAPTION_RESULT'
  payload: {
    caption: string
  }
}

export type TranscribeResultResponse = {
  id: string
  type: 'TRANSCRIBE_RESULT'
  payload: {
    vtt: string        // WEBVTT formatted subtitle string
  }
}

export type TranslateResultResponse = {
  id: string
  type: 'TRANSLATE_RESULT'
  payload: {
    arabicText: string
  }
}

export type ErrorResponse = {
  id: string           // Matches originating request ID
  type: 'ERROR'
  payload: {
    message: string
  }
}

/** Union of all valid messages sent FROM a Worker TO the main thread */
export type WorkerResponse =
  | ModelProgressResponse
  | BackendDetectedResponse
  | CaptionResultResponse
  | TranscribeResultResponse
  | TranslateResultResponse
  | ErrorResponse

// ─── Helper types ────────────────────────────────────────────────────────────

/** Extract the payload type for a given response type literal */
export type ResponsePayload<T extends WorkerResponse['type']> = Extract<
  WorkerResponse,
  { type: T }
>['payload']
```

---

## 2. Worker Instantiation — Client Component Pattern

Workers are browser-only. Instantiate them **inside a Client Component**, never at module top-level in a Server Component. Use a ref to hold the worker instance.

```tsx
// components/model-sidebar/use-vision-worker.ts
'use client'

import { useRef, useEffect, useCallback } from 'react'
import type { WorkerRequest, WorkerResponse } from '@/lib/worker-protocol'

type MessageHandler = (response: WorkerResponse) => void

export function useVisionWorker(onMessage: MessageHandler) {
  const workerRef = useRef<Worker | null>(null)

  useEffect(() => {
    // Lazy instantiation — only created when this hook mounts
    workerRef.current = new Worker(
      new URL('../../workers/vision.worker.ts', import.meta.url),
      { type: 'module' },
    )

    workerRef.current.addEventListener(
      'message',
      (event: MessageEvent<WorkerResponse>) => {
        onMessage(event.data)
      },
    )

    workerRef.current.addEventListener('error', (err) => {
      onMessage({
        id: 'unknown',
        type: 'ERROR',
        payload: { message: err.message },
      })
    })

    return () => {
      workerRef.current?.terminate()
      workerRef.current = null
    }
  }, [onMessage])

  const postRequest = useCallback((request: WorkerRequest) => {
    if (!workerRef.current) return
    // Check if payload contains a transferable buffer
    if (
      request.type === 'TRANSCRIBE' &&
      request.payload.audioBuffer instanceof Float32Array
    ) {
      // Zero-copy transfer of the underlying ArrayBuffer
      workerRef.current.postMessage(request, [
        request.payload.audioBuffer.buffer,
      ])
    } else {
      workerRef.current.postMessage(request)
    }
  }, [])

  return { postRequest }
}
```

Apply the same pattern for `useWhisperWorker` and `useTranslationWorker`.

---

## 3. Request ID Correlation

Every request-response pair is correlated by the `id` field.

### Generating request IDs
```ts
const requestId = crypto.randomUUID() // Available in all modern browsers
```

### Tracking pending requests
Use a `Map<string, PendingRequest>` in React state or a ref to track in-flight requests:

```ts
type PendingRequest = {
  resolve: (response: WorkerResponse) => void
  reject: (error: Error) => void
  timeoutHandle: ReturnType<typeof setTimeout>
}

const pendingRef = useRef(new Map<string, PendingRequest>())
```

When a response arrives, look up the `id` in the map:
```ts
const onMessage = useCallback((response: WorkerResponse) => {
  if (response.type === 'MODEL_PROGRESS' || response.type === 'BACKEND_DETECTED') {
    // Progress events — route to sidebar state, not to pending map
    handleProgressEvent(response)
    return
  }
  const pending = pendingRef.current.get(response.id)
  if (!pending) return
  clearTimeout(pending.timeoutHandle)
  pendingRef.current.delete(response.id)
  if (response.type === 'ERROR') {
    pending.reject(new Error(response.payload.message))
  } else {
    pending.resolve(response)
  }
}, [handleProgressEvent])
```

### Request timeout
Always set a timeout for requests (default: 120 seconds for large models):
```ts
const TIMEOUT_MS = 120_000

const timeoutHandle = setTimeout(() => {
  pendingRef.current.delete(id)
  reject(new Error(`Worker request ${id} timed out after ${TIMEOUT_MS}ms`))
}, TIMEOUT_MS)
```

---

## 4. Model Download Progress — Event Contract

`MODEL_PROGRESS` responses are **not** correlated to a single request ID — they are emitted during model loading and must be routed to global sidebar state, not to the pending map.

### State shape for the sidebar
```ts
// In a React context or Zustand store
type ModelStatus = {
  stage: 'idle' | 'downloading' | 'loading' | 'ready' | 'error'
  progress: number          // 0–100
  currentFile: string | null
  loadedBytes: number
  totalBytes: number
  backend: 'webgpu' | 'wasm' | null
  errorMessage: string | null
}

type ModelSidebarState = {
  vision: ModelStatus
  whisper: ModelStatus
  translation: ModelStatus
}
```

### Updating sidebar state from a progress event
```ts
function handleProgressEvent(response: WorkerResponse) {
  if (response.type === 'BACKEND_DETECTED') {
    setModelStatus(key, (prev) => ({ ...prev, backend: response.payload.backend }))
    return
  }
  if (response.type !== 'MODEL_PROGRESS') return

  const { status, file, progress = 0, loaded = 0, total = 0 } = response.payload

  setModelStatus(key, (prev) => ({
    ...prev,
    stage:
      status === 'ready'
        ? 'ready'
        : status === 'downloading'
          ? 'downloading'
          : status === 'done'
            ? 'loading'
            : prev.stage,
    progress: status === 'downloading' ? progress : prev.progress,
    currentFile: file ?? prev.currentFile,
    loadedBytes: loaded,
    totalBytes: total,
  }))
}
```

---

## 5. ARIA Live Region Integration

Every model status update that changes the sidebar must be reflected in an ARIA live region so screen reader users hear progress announcements.

```tsx
// components/model-sidebar/model-status-item.tsx
'use client'

export function ModelStatusItem({
  label,
  status,
}: {
  label: string
  status: ModelStatus
}) {
  const announcement = getAnnouncement(status)

  return (
    <div role="status" aria-label={`${label} model status`}>
      {/* Visual UI */}
      <span className="font-medium">{label}</span>
      <progress
        role="progressbar"
        aria-valuenow={status.progress}
        aria-valuemin={0}
        aria-valuemax={100}
        aria-label={`${label} model download progress`}
        value={status.progress}
        max={100}
      />
      {/* Screen-reader-only live announcement */}
      <span className="sr-only" aria-live="polite" aria-atomic="true">
        {announcement}
      </span>
    </div>
  )
}

function getAnnouncement(status: ModelStatus): string {
  switch (status.stage) {
    case 'downloading':
      return `Downloading model: ${Math.round(status.progress)}% complete`
    case 'loading':
      return 'Model downloaded. Loading into memory...'
    case 'ready':
      return `Model ready. Running on ${status.backend === 'webgpu' ? 'WebGPU' : 'CPU'}.`
    case 'error':
      return `Model failed to load: ${status.errorMessage}`
    default:
      return ''
  }
}
```

---

## 6. Zero-Copy Buffer Transfer

When sending large binary payloads (audio buffers, image `ArrayBuffer`s) to a Worker, always use the **transfer list** to avoid copying:

```ts
// Correct — zero copy, the main thread can no longer access audioBuffer.buffer after this
worker.postMessage(request, [request.payload.audioBuffer.buffer])

// Wrong — copies the entire buffer (potentially hundreds of MB)
worker.postMessage(request)
```

After a zero-copy transfer, the source `ArrayBuffer` is detached. Do not read from it in the main thread after posting.

---

## 7. TypeScript Exhaustive Switch Pattern

Always use exhaustive switches when handling `WorkerResponse` types to catch missing cases at compile time:

```ts
function handleWorkerResponse(response: WorkerResponse): void {
  switch (response.type) {
    case 'MODEL_PROGRESS':
      handleProgress(response.payload)
      break
    case 'BACKEND_DETECTED':
      handleBackend(response.payload)
      break
    case 'CAPTION_RESULT':
      handleCaption(response.id, response.payload)
      break
    case 'TRANSCRIBE_RESULT':
      handleTranscription(response.id, response.payload)
      break
    case 'TRANSLATE_RESULT':
      handleTranslation(response.id, response.payload)
      break
    case 'ERROR':
      handleError(response.id, response.payload)
      break
    default: {
      // TypeScript will error here if a new response type is added
      // but not handled above — exhaustiveness check
      const _exhaustive: never = response
      console.warn('Unhandled worker response:', _exhaustive)
    }
  }
}
```

---

## 8. Common Pitfalls

| Pitfall | Correct approach |
|---|---|
| Creating `new Worker(...)` inside a Server Component | Workers are browser-only — only instantiate in `'use client'` components inside `useEffect` |
| Not using `{ type: 'module' }` option | Required for ESM imports inside the worker (e.g., `@xenova/transformers`) |
| Routing `MODEL_PROGRESS` through the pending map | These have no corresponding pending entry — route them to sidebar state directly |
| Forgetting to `terminate()` workers on unmount | Always return a cleanup function from `useEffect` that calls `worker.terminate()` |
| Calling `postMessage` before the worker is ready | Guard with `if (!workerRef.current) return` or await a `BACKEND_DETECTED` response |
| Not transferring `Float32Array` buffers | Large audio buffers must use the transfer list — omitting it copies hundreds of MB |
