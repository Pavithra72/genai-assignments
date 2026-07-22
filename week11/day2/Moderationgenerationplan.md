# Moderationgenerationplan.md

## Objective
Integrate a Moderation layer into the visual regression application using the AI-Shield contract, with an API-first rollout.

## Mandatory Design Commitments
1. Backend API changes are implemented first.
2. Every LLM-bound request is pre-checked via POST `/api/moderate`.
3. If moderation returns `BLOCK`, no LLM call is executed.
4. Moderation integration is toggleable from environment configuration.
5. Blocked responses include:
- User-friendly guidance
- Block reason
- Detector-level details (detector name, action, reason)

## AI-Shield Contract Summary
Primary runtime integration endpoint:
1. `POST /api/moderate`

Detector endpoints available for diagnostics and validation:
1. `POST /api/detect/pii`
2. `POST /api/detect/cii`
3. `POST /api/detect/secret`
4. `POST /api/detect/toxic`
5. `POST /api/detect/injection`
6. `POST /api/detect/file`
7. `POST /api/detect/token`

Expected moderation outcomes:
1. `ALLOW`: proceed to LLM
2. `MASK`: proceed (with sanitized content if provided by moderation policy)
3. `BLOCK`: stop pipeline and return moderation-blocked response

## Phase Plan

### Phase A: API and Configuration Foundation
1. Add moderation env controls in `server/.env.example`:
- `ENABLE_MODERATION_LAYER=true|false`
- `MODERATION_API_BASE_URL=http://localhost:4000`
- `MODERATION_TIMEOUT_MS=5000`
- `MODERATION_MODEL_DEFAULT=gpt-4`
2. Parse and expose these keys in `server/src/lib/env.ts`.
3. Add moderation response and error types in `server/src/types/index.ts`.

### Phase B: Moderation Service Layer
1. Create `server/src/services/moderation.service.ts`.
2. Implement `POST /api/moderate` client:
- input: text, model, context
- output: decision + detector details
3. Add timeout and structured error mapping.
4. Behavior policy:
- If `ENABLE_MODERATION_LAYER=false`: bypass moderation
- If `ENABLE_MODERATION_LAYER=true` and moderation unavailable: fail closed for LLM-bound calls

### Phase C: Enforce Moderation Before LLM
1. Integrate guard into `server/src/services/visionAnalysis.service.ts`.
2. Integrate guard into `server/src/services/reporting.service.ts`.
3. Ensure controller paths call moderated services:
- `server/src/controllers/newFeature.controller.ts`
- `server/src/controllers/regression.controller.ts`
4. Guarantee that `BLOCK` response prevents downstream LLM invocation.

### Phase D: Error and Response Contract
1. Extend error mapping in `server/src/middleware/errorHandler.middleware.ts`.
2. Add moderation error codes:
- `MODERATION_BLOCKED`
- `MODERATION_UNAVAILABLE`
3. Standard blocked payload should include:
- `userMessage`
- `reason`
- `detectorDetails` array with detector, action, reason

### Phase E: API Contract Exposure
1. Optionally expose moderation-enabled flag in `server/src/controllers/config.controller.ts` response for UI.
2. Keep existing API envelopes backward-compatible.

### Phase F: Postman Coverage
1. Extend `postman/visual-regression-tool.postman_collection.json` with moderation-aware requests:
- new-feature allowed
- new-feature blocked
- regression allowed
- regression blocked
2. Keep `AI-Shield.postman_collection.json` as detector reference collection.

### Phase G: UI Integration (After API Completion)
1. In client, handle moderation-blocked responses globally.
2. Show:
- concise blocked message
- detector details
- remediation guidance (remove sensitive data, rephrase prompt, contact admin)
3. Preserve normal flow for `ALLOW` and `MASK` results.

## Implementation Scope Boundaries
Included:
1. Moderation pre-check integration for all LLM-bound application paths
2. Env-toggle support
3. Detector-detail propagation to API and UI

Excluded:
1. Rebuilding AI-Shield detector internals
2. Data persistence/caching of moderation decisions

## Verification Plan
1. Compile backend successfully.
2. `ENABLE_MODERATION_LAYER=false`:
- existing flows unchanged
3. `ENABLE_MODERATION_LAYER=true` and moderate=`ALLOW`:
- LLM path executes
4. `ENABLE_MODERATION_LAYER=true` and moderate=`BLOCK`:
- LLM path does not execute
- API returns moderation-blocked response with detector details
5. Moderation timeout/downstream failure:
- fail-closed behavior confirmed on LLM-bound endpoints
6. UI displays blocked guidance and detector details correctly.
7. Postman runs validate allowed and blocked behavior for new-feature and regression paths.

## Recommended Execution Order
1. Phase A
2. Phase B
3. Phase C
4. Phase D
5. Phase E
6. Phase F
7. Phase G
