# Langfuse Tracing Architecture — `/api/search`

## Purpose

This document describes the design for adding [Langfuse](https://langfuse.com) observability to the RAG search API. The goal is visibility into two things that are currently opaque in logs only: the cost/latency of the Mistral embedding call, and the behavior (inputs, filters, result scores) of the MongoDB Atlas `$vectorSearch` call.

**Scope:** `POST /api/search`, `POST /api/search/bm25`, and `POST /api/search/hybrid` are instrumented. Other routes remain out of scope for this pass.

## Components touched

| Component | Role |
|---|---|
| [server/index.js](server/index.js) | Singleton Langfuse client is created once at module load, near the existing `dotenv.config()` call ([server/index.js:19](server/index.js#L19)). |
| [server/index.js:1781-1919](server/index.js#L1781-L1919) — `POST /api/search` handler | Existing Vector trace implementation used as the reference pattern. |
| [server/index.js](server/index.js) — `POST /api/search/bm25` handler | BM25 trace creation, BM25 pipeline span, trace output update, and success/error flush. |
| [server/index.js](server/index.js) — `POST /api/search/hybrid` handler | Hybrid trace creation, embedding generation span, BM25 span, vector span, fusion span, trace output update, and success/error flush. |
| [src/scripts/utilities/mistralEmbedding.js](src/scripts/utilities/mistralEmbedding.js) — `generateEmbedding` | The call wrapped as a generation span (no changes needed inside this file — it's wrapped from the call site). |

## Trace structure

One trace per request to each instrumented route:

```
trace: "search"
  input: { query, filters }
  │
  ├── generation: "generate-embedding"
  │     model: "mistral-embed"
  │     input: query
  │     output: { tokens, cost }
  │     (wraps generateEmbedding(query) at server/index.js:1811)
  │
  └── span: "vector-search"
        input: aggregation pipeline (vector stage + filters + limit)
        output: results + scores
        (wraps collection.aggregate(pipeline).toArray() at server/index.js:1896)
  │
  output: final `results` array (same shape returned to the client)
```

```
trace: "search-bm25"
  input: { query, filters }
  │
  └── span: "bm25-search"
        input: BM25 `$search` aggregation pipeline
        output: BM25 results + scores
  │
  output: final `results` array (same shape returned to the client)
```

```
trace: "search-hybrid"
  input: { query, filters, weights }
  │
  ├── generation: "generate-embedding"
  │     model: "mistral-embed"
  │     input: query
  │     output: { tokens, cost }
  │
  ├── span: "bm25-search"
  │     input: BM25 pipeline
  │     output: BM25 candidate results
  │
  ├── span: "vector-search"
  │     input: vector pipeline
  │     output: vector candidate results
  │
  └── span: "hybrid-fusion"
        input: { limit, filters, weights }
        output: final count + source-distribution stats
  │
  output: final `results` array (same shape returned to the client)
```

- The trace's `output` is set right before `res.json(responseData)`, using the same `results` payload sent to the client — no separate serialization.
- The generation span records the token usage and cost that the handler already computes (`embeddingResult.usage`, `cost`) — no new cost-calculation logic is introduced.
- The vector-search span's `input` is the pipeline array already built in the handler ([server/index.js:1840](server/index.js#L1840)) — logged as-is, not reshaped.
- BM25 and Hybrid routes follow the same lifecycle pattern as Vector: `trace(...)` → route-specific spans/generation → `trace.update({ output })` → `flushAsync()` on success and on error path.

## Client setup

- Add dependency: `npm install langfuse`
- New environment variables (added to `.env`, no real values committed):
  - `LANGFUSE_PUBLIC_KEY`
  - `LANGFUSE_SECRET_KEY`
  - `LANGFUSE_HOST`
- Client is a **singleton**: instantiated once at module load in `server/index.js`, imported/reused by the request handler — not re-created per request.

## Flush strategy

Langfuse batches events internally, so each handler explicitly calls:

```js
await langfuse.flushAsync();
```

- Once on the success path, right before `res.json(responseData)`.
- Once in the `catch` block, so failed requests still show up in the dashboard.

This is deliberate — the implementation does **not** rely on process-shutdown flush, since a single long-running Express process shouldn't be the only thing guaranteeing traces are sent, and future serverless/short-lived deployments would silently drop unflushed events.

## Out of scope (deliberate YAGNI cuts)

- **No generic tracing wrapper/helper** — `langfuse.trace()/span()/generation()` are called inline in the handler, not through an abstraction layer.
- **No config flag to toggle tracing on/off** — tracing is always on when the route is hit.
- **No batching or sampling logic** — every request is traced.

These may be revisited later if a second route needs the same instrumentation and duplication becomes a real problem — not before.

## Verification

Once implemented:
1. Start the server (`npm run server`).
2. Send sample requests:
  - `POST /api/search` with `{ "query": "...", "limit": 5 }`
  - `POST /api/search/bm25` with `{ "query": "...", "limit": 5 }`
  - `POST /api/search/hybrid` with `{ "query": "...", "limit": 5, "bm25Weight": 0.5, "vectorWeight": 0.5 }`
3. Open the Langfuse dashboard and confirm:
  - Traces named `search`, `search-bm25`, and `search-hybrid` appear for their corresponding routes.
  - `search-bm25` contains a `bm25-search` span with pipeline input and scored output.
  - `search-hybrid` contains `generate-embedding`, `bm25-search`, `vector-search`, and `hybrid-fusion` observations.
  - Trigger an error case and confirm traces still appear via the `catch`-block flush on BM25 and Hybrid handlers.
