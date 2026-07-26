# Langfuse Tracing — README

Observability for `POST /api/search`, `POST /api/search/bm25`, and `POST /api/search/hybrid`, added via [Langfuse](https://langfuse.com). This document covers only the Langfuse integration — flow, functionality, every file it touches, and its `.env` configuration. For the design rationale, see [Architecture.md](Architecture.md).

## Functionality

Every call to the instrumented search routes produces one Langfuse **trace** with route-specific **observations**:

- `POST /api/search` (`search`): generation span for embeddings + span for vector pipeline.
- `POST /api/search/bm25` (`search-bm25`): span for BM25 pipeline.
- `POST /api/search/hybrid` (`search-hybrid`): generation span for embeddings + BM25 span + vector span + fusion span.

Each trace carries a `userId` — a per-browser anonymous ID — so requests can be correlated back to a browser session in the Langfuse dashboard, and the trace's top-level `input`/`output` mirror what the client sent and received.

Non-search routes remain uninstrumented.

## Flow

```
Browser (search pages + processing flows)
  │  generates/reuses an anonymous userId (localStorage)
  │  POST /api/search*  { query, limit, filters/weights, userId }
  ▼
server/index.js — route handler
  │
  ├─ langfuse.trace({ name, userId, input })
  │
  ├─ trace.generation(...) where route requires embeddings
  ├─ trace.span(...) for BM25/vector/fusion stages (route dependent)
  │     await collection.aggregate(...).toArray()
  │     .end({ output })
  │
  ├─ trace.update({ output: results })
  ├─ res.json(responseData)
  └─ await langfuse.flushAsync()
        (also runs in the catch block, so failed requests are traced too)
```

Flushing is explicit and awaited on both the success and error paths — the SDK's internal batching is never left to flush itself on process shutdown.

## Impacted files

| File | What changed |
|---|---|
| [package.json](package.json#L23) | Added `"langfuse": "^3.38.20"` dependency. |
| [.env](.env#L21-L24) | Added `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`, `LANGFUSE_HOST`. |
| [server/index.js:15](server/index.js#L15) | `import { Langfuse } from 'langfuse'`. |
| [server/index.js:22-26](server/index.js#L22-L26) | Singleton `langfuse` client, constructed once at module load from the env vars above. |
| [server/index.js:1788-1953](server/index.js#L1788-L1953) | The `POST /api/search` handler — trace creation, the `generate-embedding` generation, the `vector-search` span, trace output, and the two `flushAsync()` calls. Relevant lines: trace creation ([1790-1796](server/index.js#L1790-L1796)), generation ([1818-1840](server/index.js#L1818-L1840)), span ([1915-1917](server/index.js#L1915-L1917)), flush ([1932-1952](server/index.js#L1932-L1952)). |
| [server/index.js](server/index.js) | `POST /api/search/bm25` handler now includes `search-bm25` trace, `bm25-search` span, trace output update, and success/error-path flush. |
| [server/index.js](server/index.js) | `POST /api/search/hybrid` handler now includes `search-hybrid` trace, `generate-embedding` generation, `bm25-search` span, `vector-search` span, `hybrid-fusion` span, trace output update, and success/error-path flush. |
| [client/src/components/search/QuerySearch.js:37-47](client/src/components/search/QuerySearch.js#L37-L47) | `getAnonymousUserId()` — creates/reuses a `crypto.randomUUID()` stored in `localStorage`. |
| [client/src/components/search/QuerySearch.js:119-124](client/src/components/search/QuerySearch.js#L119-L124) | The `axios.post` call to `/api/search` now includes `userId: getAnonymousUserId()` in the request body. |
| [client/src/utils/anonymousUserId.js](client/src/utils/anonymousUserId.js) | Shared anonymous user ID helper used by BM25/Hybrid and processing flows. |
| [client/src/components/search/BM25Search.js](client/src/components/search/BM25Search.js) | BM25 request now includes `userId`. |
| [client/src/components/search/HybridSearch.js](client/src/components/search/HybridSearch.js) | Hybrid request now includes `userId`. |
| [client/src/components/processing/QueryPreprocessing.js](client/src/components/processing/QueryPreprocessing.js) | Search workflow request now includes `userId`. |
| [client/src/components/processing/SummarizationDedup.js](client/src/components/processing/SummarizationDedup.js) | Search workflow request now includes `userId`. |
| [client/src/components/processing/PromptSchemaManager.js](client/src/components/processing/PromptSchemaManager.js) | Hybrid and user-stories hybrid workflow requests now include `userId`. |

## `.env` configuration

Added to [.env](.env), directly below the existing Groq config block:

```
# Langfuse Configuration (for tracing /api/search*)
LANGFUSE_PUBLIC_KEY=<your Langfuse project public key, starts with pk-lf-...>
LANGFUSE_SECRET_KEY=<your Langfuse project secret key, starts with sk-lf-...>
LANGFUSE_HOST=<your Langfuse instance URL, e.g. https://cloud.langfuse.com>
```

These three are read once at server startup ([server/index.js:22-26](server/index.js#L22-L26)) to construct the singleton client — changing them requires restarting the Node process (`node server/index.js` has no hot-reload).

## Verifying it's working

1. Start the server and call the three endpoints (`/api/search`, `/api/search/bm25`, `/api/search/hybrid`) with a `userId` in the request body.
2. Server logs should show `✅ Langfuse trace flushed: <trace-id>` for success, and `✅ Langfuse trace flushed (error path)` for failures.
3. In the Langfuse dashboard, open **Traces** and verify:
  - `search` traces include `generate-embedding` and `vector-search`.
  - `search-bm25` traces include `bm25-search`.
  - `search-hybrid` traces include `generate-embedding`, `bm25-search`, `vector-search`, and `hybrid-fusion`.
