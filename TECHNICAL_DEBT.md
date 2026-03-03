# Technical debt

Known design flaws, compromises, and their impact. No speculation beyond what the code and structure justify.

---

## 1. All API state is in-memory

**What:** Sessions, answers, interview state (coverage, questionToDimension, askedQuestionTitles), assessment summaries, and report cache are stored in process memory (Map structures in sessionService, assessmentService, reportService).

**Why:** Simplifies initial implementation; no database or shared store is required.

**Impact:** State is lost on process restart. Multiple API instances do not share state; session id from one instance is not visible to another. No durability for audit, resume, or multi-device use.

**Assessment:** Unacceptable for production multi-instance or durable workloads. Acceptable for single-instance development or demo.

**Priority:** High if persistence or horizontal scaling is required.

---

## 2. Optional interview config uses code defaults

**What:** In assessmentService, `getInterviewConfig()` uses defaults when MIN_SIGNAL_PER_DIMENSION or MAX_INTERVIEW_QUESTIONS are unset: minSignal = 1, maxQuestions = undefined. The workspace rule forbids defaults in code for required config; these are optional and documented in env.example.

**Why:** Allows the interview to run without setting these env vars; “sensible” defaults keep the flow working.

**Impact:** Minor. Behavior is consistent but differs from the strict “no defaults in code” rule for optional numeric/limits config. Documented in env.example as optional.

**Assessment:** Acceptable as documented optional behavior. If the project enforces “no defaults anywhere,” these should be required or moved to a dedicated config file.

**Priority:** Low.

---

## 3. Backward-compatibility shim for Ollama config

**What:** `config/ollama.js` re-exports and reshapes config from `config/llm.js` (e.g. mode, enabled) for “backward compatibility.” README and llm.js are the canonical source; ollama.js is a thin wrapper.

**Why:** Existing or external code may require the old ollama.js shape.

**Impact:** Two entry points for LLM config; risk of drift if one is updated and the other is not. Currently both read from llm.js, so behavior is consistent.

**Assessment:** Acceptable as long as ollama.js is not extended. Prefer migrating callers to llm.js and deprecating ollama.js.

**Priority:** Low unless removing legacy callers.

---

## 4. Pre-survey optional question IDs hardcoded in UI

**What:** In PreSurveyWizard, `OPTIONAL_QUESTION_IDS = new Set([1, 2, 5])` (gender, age/stage, risk) is hardcoded. Adding or reordering questions in pre_survey_questions.json requires code change to keep “optional” behavior correct.

**Why:** Quick way to mark which steps can be skipped without changing the JSON schema.

**Impact:** Coupling between question id and code; schema changes require code updates. No single source of truth in data for “optional” flag.

**Assessment:** Acceptable for current small set. Unacceptable if pre-survey schema evolves often; then optionality should live in data (e.g. optional: true per question).

**Priority:** Low unless pre-survey structure is extended.

---

## 5. Report template vs actual report payload

**What:** reportStructure.js defines REPORT_SECTIONS (e.g. strengthProfileSummary, careerClusterAlignment). The actual report object includes additional keys (e.g. strengthProfileSummaryLLM, strengthProfileSummaryHybrid, profileByDimensions) that are not in REPORT_SECTIONS. Template initializes sections to null; reportService overlays synthesis results and _meta.

**Why:** REPORT_SECTIONS reflects the product list of sections; implementation added distinct keys for LLM vs hybrid summary and for structured profile data.

**Impact:** REPORT_SECTIONS is not the full set of keys the UI or consumers use. Iterating REPORT_SECTIONS would miss LLM/hybrid-specific keys. Risk of confusion when adding new sections.

**Assessment:** Acceptable if documented (as in CONTRACTS.md). Unacceptable if tooling or docs assume REPORT_SECTIONS is the complete contract. Consider aligning names or documenting “synthetic” keys separately.

**Priority:** Low; document and keep consistent.

---

## 6. Duplicate session-id handoff logic in UI

**What:** RESULTS_SESSION_KEY and the logic to resolve sessionId from location.state or sessionStorage are duplicated in ResultsPage and RecommendationsPage. Same constant and same fallback order in two files.

**Why:** Each page was implemented to work standalone when opened directly or after navigation.

**Impact:** Changes to key name or resolution order must be done in two places; risk of drift.

**Assessment:** Acceptable. Fix by extracting a small helper or shared constant/hook.

**Priority:** Low.

---

## 7. No explicit validation of API request bodies

**What:** Controllers pass req.body (or parts of it) to services with minimal validation. Session create accepts body.id and body.preSurveyProfile; assessment submitAnswers accepts payload.answers or payload as single answer. Services tolerate missing or malformed data with fallbacks (e.g. empty array, null).

**Why:** Keeps controllers thin and avoids duplicating validation before it was needed.

**Impact:** Invalid or malicious payloads can lead to inconsistent state or confusing behavior. No single place that documents and enforces request shape.

**Assessment:** Acceptable for trusted or internal use. Unacceptable for public or strict API contracts; add validation (e.g. schema or explicit checks) and return 400 with clear errors.

**Priority:** Medium if API is exposed to untrusted clients.

---

## 8. Report cache invalidation only by answer count

**What:** Report cache (reportService) is keyed by sessionId and answerCount. Any change that should invalidate the report (e.g. session status, or future edits to answers) does not currently affect the cache key.

**Why:** Answer count is the only changing input to synthesis in the current design; simple and sufficient for “one report per session state.”

**Impact:** If session metadata or answer content could change without the count changing, cache could serve a stale report. Not an issue with current flow (append-only answers).

**Assessment:** Acceptable for current append-only model. Revisit if answers or session become editable.

**Priority:** Low unless product adds edit/update flows.

---

## 9. Error handling and logging

**What:** LLM and report synthesis catch errors and log; many return null or fallback (e.g. static questions, no recommendations). Controllers rely on a global error handler for thrown errors. No structured error codes or machine-readable error types for clients.

**Why:** Keeps the app running when LLM or optional steps fail; avoids leaking internals.

**Impact:** Clients see generic messages; debugging depends on server logs. Retry or partial-success behavior is not standardized.

**Assessment:** Acceptable for MVP. Unacceptable for strict SLA or client-driven retry logic; then add error codes and consistent response shape.

**Priority:** Medium if API consumers need to distinguish failure types.

---

## Priority order for addressing

1. **High (if requirements demand):** In-memory state (persistence and/or shared store for sessions, assessment, report).
2. **Medium (if API is public or strict):** Request body validation; structured error handling and codes.
3. **Low:** Optional interview config defaults; ollama.js backward-compat; pre-survey optional IDs; report template vs payload documentation; duplicate session-id logic in UI; report cache key design; error message consistency.
