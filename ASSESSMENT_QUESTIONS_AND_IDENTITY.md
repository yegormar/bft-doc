# Assessment questions, pre-generation, and user identity

This document describes how the API serves assessment questions (session queue, question generation component, permanent store), the browser identity cookie, and per-user “already used” question tracking.

---

**See also:** [Scenario batches, design instructions, and rank questions](SCENARIO_BATCHES_AND_RANK.md) — batch-based dimension selection, scenario design prompt, and the rank question type.

## Browser identity cookie (`bft_uid`)

- **Purpose:** Identify the same browser/user across sessions so the server can avoid reusing questions this user has already seen (when pulling from the permanent store).
- **Name:** `bft_uid`
- **Value:** A ULID (26-character string), generated on the first API request if the cookie is missing or invalid.
- **Lifetime:** 1 year (`maxAge`).
- **Options:** `httpOnly: true`; in production `sameSite: 'none'` and `secure: true` (for cross-origin); in development `sameSite: 'lax'`.
- **Where set:** API middleware runs for all `/api` requests; if `bft_uid` is absent or not a valid ULID, the server generates a new one and sets it on the response. The value is attached to `req.bftUserId` for controllers and services.
- **Client:** The UI must send credentials so the cookie is sent and received: `fetch(..., { credentials: 'include' })`. The API CORS config uses `credentials: true` so the browser will send the cookie on same- or cross-origin requests when the frontend uses `credentials: 'include'`.

---

## Question store (permanent, file-based)

- **Purpose:** Persist generated questions by profile so they survive restarts and can be reused for other users with the same profile.
- **Config:** Optional env var `BFT_QUESTIONS_STORE_DIR`. If unset, questions are not persisted and store reuse is disabled; “used” tracking is in-memory only for the lifetime of the process.
- **Layout:**
  - `{BFT_QUESTIONS_STORE_DIR}/{profileKey}/{createdAt}_{contentHash}.json` — one file per question. `createdAt` is ISO with colons replaced by `-` for portability. `contentHash` is the first 16 chars of the SHA-256 of the question content (title + option texts).
  - `{BFT_QUESTIONS_STORE_DIR}/used/{bftUserId}.json` — array of content hashes that this user has already been served (so we do not reuse them).
- **Profile key:** Derived from `preSurveyProfile` (canonical JSON with sorted keys, then SHA-256 hex, first 16 chars). Same profile always yields the same key so questions are shared across users with the same profile.
- **Write:** When the API generates a new question via the LLM (request path or background pre-generation), it saves that question to the store under the session’s profile key.
- **Read:** When the session’s pre-generated queue is empty, the API lists questions for that profile key, excludes any whose content hash is in this user’s “used” set, sorts by `createdAt` ascending (oldest first), and takes the first. So “recent” (newer) questions stay at the back of the line.

---

## Question store directory structure

Root directory is set by **`BFT_QUESTIONS_STORE_DIR`** (e.g. `./data/questions-store`). If unset, the store is disabled.

### Directory tree

```
{BFT_QUESTIONS_STORE_DIR}/
├── {profileKey}/                    ← one subdir per pre-survey profile (hash)
│   ├── {createdAt}_{contentHash}.json
│   ├── {createdAt}_{contentHash}.json
│   └── ...
└── used/                            ← per-user "already served" tracking
    ├── {bftUserId}.json
    └── ...
```

### Profile subdirs: `{storeDir}/{profileKey}/`

- **profileKey** = first 16 hex characters of SHA-256 of the **canonical** pre-survey profile (JSON with sorted keys at every level). Same profile always yields the same key; different users with the same profile share this directory.
- **One file per stored question** in that profile.
- **Filename:** `{createdAt}_{contentHash}.json`
  - **createdAt** = ISO 8601 timestamp with colons replaced by `-` (e.g. `2025-03-03T21-38-32.412Z`) for safe filenames.
  - **contentHash** = first 16 hex characters of SHA-256 of the question content. Content = question title + newline + sorted option texts (one per line). Used for deduplication and "used" tracking.
- **File content (JSON):**
  - `question` — `{ title, description?, type, options }`
  - `dimensionSet` — array of dimensions this question targets (`dimensionType`, `dimensionId`, etc.)
  - `assessmentSummary` — string or null
  - `profileKey`, `createdAt`, `contentHash` — metadata

Example: `questions-store/44039c7ae9c64502/2025-03-03T21-38-32.412Z_b5b43dd6a1af.json`

### Used tracking: `{storeDir}/used/{bftUserId}.json`

- **bftUserId** = browser identity from cookie (ULID). One file per user.
- **File content:** JSON array of **full** content hashes (full SHA-256 hex strings) of every question already served to that user. Used so the same user is never given the same question again (even across sessions).
- When a question is served (from session queue, store, or LLM), its content hash is appended to this array (and the file is rewritten).

### Summary

| Path | Purpose |
|------|--------|
| `{storeDir}/{profileKey}/*.json` | Stored questions for that profile; filename = `createdAt_contentHash16.json`. |
| `{storeDir}/used/{bftUserId}.json` | Content hashes of questions already served to that user (no reuse). |

---

## Question generation component

All "generate one question" work goes through a dedicated component (`bft-api/src/services/questionGeneration/`). It:

- **FIFO queue:** Requests from different sessions are processed one at a time, so concurrent users do not all hit the LLM in parallel.
- **Default:** When a question is needed, the request is sent to the LLM (after waiting for its turn in the queue).
- **Timeout:** If the LLM does not respond within `BFT_QUESTION_LLM_TIMEOUT_MS` (required in .env when using assessments), the component falls back to the store.
- **LLM failure:** If the LLM throws or returns invalid output, the same store fallback is used.
- **Store fallback:** From the permanent store (same profile key), the component excludes questions already used for this cookie (`bftUserId`), scores remaining candidates by how well their dimension set matches the desired dimensions (traits/attributes being measured), and returns the best match (tie-break: oldest first). If no candidate remains, the component returns `null` and the API uses the fixed `MAIN_QUESTIONS` fallback.

**Config:** `BFT_QUESTION_LLM_TIMEOUT_MS` must be set in .env (e.g. 15000–30000 ms). See `bft-api/env.example`.

---

## Full question generation logic

This section walks through how the next question is chosen and how the question generation component works.

### 1. Entry point: `getNextQuestion(sessionId, bftUserId)` (assessmentService)

When the API needs the next question for a session:

1. **Interview complete?** If coverage and config say the interview is done → return `completed: true`, no question.
2. **Session pre-generated queue:** If this session has at least one pre-generated question in memory, **pop one**, apply it to session state, mark as used for `bftUserId`, return it. Optionally trigger background refill if queue size is below threshold.
3. **Wait for pregen:** If the queue is empty but background pre-generation is still running for this session, **wait up to 25 s** (poll every 300 ms). If a question appears in the queue, pop it and return as in step 2. If timeout or pregen finishes with nothing, continue.
4. **Need a new question:** Compute **desired dimension set** (which dimensions to measure next) via `selectNextDimensionSet(state.coverage, model)`. Then call the **question generation component** (below). If the component returns a result → apply to state, **save to store only if source was LLM**, mark as used, trigger background pregen, return. If the component returns **null** → use the fixed **MAIN_QUESTIONS** fallback (by index), apply to state, optionally save that fallback to the store, trigger pregen, return.

### 2. Question generation component: `requestQuestion(context)` (FIFO → processor)

Single entry point: **`requestQuestion(context)`**. Context includes: `sessionId`, `bftUserId`, `preSurveyProfile`, `storeDir`, `desiredDimensionSet`, `askedQuestionTitles`, `answers`.

- **FIFO queue:** The call is **enqueued**. A single worker processes one request at a time. So many concurrent sessions share one queue and do not all hit the LLM in parallel.
- **Processor** (when this request's turn comes):

  1. **LLM with timeout (if LLM enabled):**  
     Call `generateScenarioQuestionWithTimeout(desiredDimensionSet, askedQuestionTitles, answers, preSurveyProfile, timeoutMs, ollamaInterview.generateScenarioQuestion)`.  
     - **Timeout:** `timeoutMs` comes from `BFT_QUESTION_LLM_TIMEOUT_MS`. The LLM call is raced against a timer; if the timer wins, the function returns `null`.  
     - **LLM error:** If the LLM throws or returns invalid data, the function returns `null`.  
     - **Success:** If the result has a valid `nextQuestion` with a `title`, return `{ question, dimensionSet: desiredDimensionSet, assessmentSummary, source: 'llm' }`.

  2. **Store fallback (if LLM returned null or skipped):**  
     - `profileKey = getProfileKey(preSurveyProfile)`.  
     - `usedSet = getUsedSet(storeDir, bftUserId)` (content hashes already served to this user).  
     - `candidates = listByProfile(storeDir, profileKey)` (all stored questions for this profile).  
     - **selectBestFromStore(candidates, usedSet, desiredDimensionSet):**  
       - Filter out any candidate whose `contentHash` is in `usedSet`.  
       - For each remaining candidate, **score** = number of its `dimensionSet` entries that appear in `desiredDimensionSet` (match on `dimensionType` + `dimensionId`).  
       - Pick the candidate with **highest score**. Tie-break: **oldest first** (`createdAt` ascending).  
       - Return `{ question, dimensionSet, assessmentSummary }` or `null` if no candidate.  
     - If a best match is found, return `{ ..., source: 'store' }`.  
     - If `storeDir` is null or no candidate remains, return **null**.

  3. **Caller (assessmentService)** treats `null` as "use MAIN_QUESTIONS fallback".

### 3. After a question is returned

- **Apply to state:** Increment question index, update coverage, record question–dimension mapping, append to asked titles. **Mark as used:** add the question's content hash to this user's used set (and persist under `used/{bftUserId}.json` when store is enabled).
- **Save to store (only when source is LLM):** Persist the question under `{storeDir}/{profileKey}/{createdAt}_{contentHash}.json` so it can be reused later for the same or other users with that profile.
- **Background pre-generation:** For this session, in the background, simulate an answer to the current question and repeatedly call the question generation component (same FIFO, timeout, store fallback) to fill the session's pre-generated queue up to a cap (and save new LLM questions to the store). One pregen run per session at a time.

### 4. Summary flow (high level)

```
getNextQuestion
  → session queue has item? → pop, apply, mark used, return
  → pregen running? → wait up to 25s → if item appears, pop and return
  → requestQuestion(context)  [FIFO]
       → LLM with timeout → success? → return { source: 'llm' }
       → else: store fallback (best match by dimensions, exclude used for user) → return { source: 'store' } or null
  → result ? apply, save if LLM, mark used, pregen, return
  → else: MAIN_QUESTIONS[index], apply, optionally save, pregen, return
```

---

## Per-user “already used” questions and reverse time order

- **Goal:** The same user (same `bft_uid`) must not see the same question again. When reusing from the permanent store, questions are served in **reverse time order**: oldest first, so the most recently stored questions are at the back.
- **Identifier:** Each question is identified by a **content hash** (SHA-256 of title + sorted option texts). The same question content is treated as one regardless of source (session queue, store, or LLM).
- **When marked as used:** Whenever a question is **served** to the user (from session queue, permanent store, or LLM), its content hash is added to that user’s used set (keyed by `bftUserId`) and persisted under `used/{bftUserId}.json` when the store is enabled.
- **Session queue:** Questions in the session queue were generated for this session and are consumed as-is. When we serve one, we still mark it as used for `bftUserId` so that if the same user starts a new session later, we will not reuse that question from the permanent store.

---

## Flow: where the next question comes from

1. If the interview is complete → return `completed: true`.
2. If the session’s **pre-generated queue** has at least one item → pop one, apply it to session state, mark as used for `bftUserId`, return it. Optionally trigger a background refill if the queue size drops below a threshold.
3. Else, if background pre-generation is **still running**, wait up to 25 seconds (polling every 300 ms) for the queue to get at least one item. If one appears, return it as in step 2.
4. Else → call the **question generation component** with the desired dimension set. The component (FIFO) tries the LLM with timeout, then store fallback (best match by dimensions, excluding used for this user). If the component returns a question, apply to state, save to store if source was LLM, mark as used, trigger background pre-generation, return it. If the component returns `null`, use the fixed **MAIN_QUESTIONS** fallback, apply to state, save to store if enabled, trigger pre-generation, return it.

Background pre-generation runs once per session at a time and also uses the question generation component: it simulates that the user answered the current question (e.g. with the first option value), then repeatedly requests questions via the component (FIFO, LLM timeout, store fallback) and pushes them to the session queue (and saves to the store when source is LLM) until the queue reaches a cap or the interview would be complete.

---

## API logging (`[bft]`)

All API-level events use the `[bft]` prefix so you can grep or filter logs easily.

| Log pattern | Meaning |
|-------------|--------|
| `[bft] cookie new bft_uid=...` | New browser identity cookie was created (first request or invalid cookie). |
| `[bft] session created sessionId=... hasProfile=true\|false` | New session created; `hasProfile` indicates whether a pre-survey profile was sent. |
| `[bft] session preSurveyProfile sessionId=... profile=...` | Session created with a pre-survey profile (JSON). |
| `[bft] pregen depth queueCap=... refillThreshold=...` | At startup: pre-generation depth (from `BFT_PREGEN_QUEUE_CAP`, `BFT_PREGEN_REFILL_THRESHOLD`). |
| `[bft] pregen disabled (BFT_PREGEN_QUEUE_CAP=0)` | At startup: background pre-generation is disabled. |
| `[bft] store enabled dir=...` | Question store is enabled at startup (`BFT_QUESTIONS_STORE_DIR` set). |
| `[bft] store disabled (...)` | Question store is disabled at startup (env unset). |
| `[bft] store saved profileKey=... contentHash=... title="..."` | A question was written to the persistent store (after LLM or background pregen). |
| `[bft] store lookup no_unused sessionId=...` | Store was checked but no unused question for this profile/user (empty or all used). |
| `[bft] question source=session_queue sessionId=... questionId=... queueRemaining=...` | Next question was served from the in-session pre-generated queue. |
| `[bft] question source=store sessionId=... questionId=...` | Next question was taken from the persistent store (best match by dimensions, via question generation component). |
| `[bft] question source=llm sessionId=... questionId=...` | Next question was generated by the LLM (via question generation component). |
| `[bft] question source=fallback sessionId=...` | Question generation component returned null; fixed MAIN_QUESTIONS fallback was used. |
| `[bft] assessment complete sessionId=... questionsAsked=...` | Interview is complete for this session. |
| `[bft] answers submitted sessionId=... count=... total=...` | User submitted one or more answers. |
| `[bft] background_pregen started sessionId=...` | Background pre-generation of questions started for this session. |
| `[bft] background_pregen finished sessionId=... queueSize=...` | Background pre-generation finished; `queueSize` is the number of questions now in the queue. |
| `[bft] background_pregen skipped sessionId=... (already running)` | A new pregen run was skipped because one is already in progress for this session. |
| `[bft] llm error sessionId=... using fallback err=...` | (Legacy log; component now handles LLM failure and store fallback.) |
| `[bft] store save failed ...` | Writing a question to the store failed (e.g. disk full). |
