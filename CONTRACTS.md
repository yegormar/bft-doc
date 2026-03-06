# Contracts

This document describes the purpose, public interfaces, invariants, and stability of each major module. Interfaces are conceptual; exact field names and types are implied by the code and API behavior.

---

## bft-api

### Config (`config/index.js`, `config/llm.js`)

- **Purpose:** Provide server and LLM configuration from environment. Ensure no required value is missing or invalid at startup.
- **Public interface:** Exported object with `port`, `nodeEnv`, `corsOrigin` (index); LLM object with `provider`, `baseUrl`, `apiKey`, `model`, `temperature`, `maxTokens`, `topP`, `systemPromptFile`, getters for `enabled`, `getSystemPrompt()`, and report-related prompt getters (llm). Optional report prompt env vars; if set, file must exist.
- **Invariants:** Process exits on missing/invalid required env. No code defaults for required vars. PORT in 1–65535; NODE_ENV in allowed set; CORS_ORIGIN non-empty; LLM provider ollama or ollama_cloud; prompt file paths must point to existing files when required.
- **Internal vs external:** Config may change which env vars are required or add new optional ones. Callers must not assume default values for any required key.
- **Stability:** Stable. Adding new optional env is allowed; removing or changing semantics of required vars is breaking.

---

### Session service (`src/services/sessionService.js`)

- **Purpose:** Create, read, and update sessions in memory. Session is the root entity for assessment and report.
- **Public interface:** `create(preSurveyProfile, clientId)` → session; `getById(id)` → session or null; `update(id, updates)` → session or null. Session: `id`, `status`, `createdAt`, `updatedAt`, `preSurveyProfile` (optional). Status one of: draft, in_progress, completed.
- **Invariants:** Session id is a valid ULID (26 chars, Crockford base32). If clientId is provided and valid ULID, it is used as id; otherwise a new ULID is generated. At most one session per id in the Map.
- **Internal vs external:** Storage is in-memory Map; may be replaced with a persistent store without changing the public interface. Callers must not rely on process lifetime.
- **Stability:** Evolving. Adding optional session fields is allowed; changing id format or required fields is breaking.

---

### Assessment service (`src/services/assessmentService.js`)

- **Purpose:** Store answers per session, maintain interview state (coverage, asked questions, dimension mapping), determine next question (LLM or static fallback), and expose assessment summary and progress.
- **Public interface:** `submitAnswers(sessionId, payload)` → array of answers; `getAssessment(sessionId)` → assessment object (sessionId, answers, assessmentSummaries, coverageSummary, insights); `getNextQuestion(sessionId)` → `{ completed, nextQuestion, assessmentSummary?, progress }`. Progress: `questionsAsked`, `coveredDimensions`, `totalDimensions`, `percentComplete`.
- **Invariants:** Answers are appended; each answer can be tied to dimensions via `questionToDimension`. Coverage is per dimension (aptitudes, traits, values, skills) with at least `questionCount` per dimension. Interview is complete when every dimension has at least `minSignalPerDimension` (config) or when `maxQuestions` (config) is reached. Next question has `id`, `title`, `type` (single_choice | multi_choice), `options`; id is assigned server-side (e.g. scenario_N). Optional env MIN_SIGNAL_PER_DIMENSION, MAX_INTERVIEW_QUESTIONS control completion; internal defaults used when not set.
- **Internal vs external:** Coverage keys (aptitudes, traits, values, skills) and dimension types (aptitude, trait, value, skill) are part of the assessment model; changing them affects report and LLM. Callers (controllers, reportService) depend on getAssessment and getNextQuestion shape.
- **Stability:** Evolving. Adding optional fields to progress or nextQuestion is allowed. Changing completion rules or dimension set is breaking for UI and report.

---

### Report service (`src/services/reportService.js`)

- **Purpose:** Build the report for a session from template, assessment data, and LLM synthesis. Cache by session and answer count; recompute when answer count changes.
- **Public interface:** `getReport(sessionId)` → report object. Report includes: `sessionId`, `status`, all keys from report template (see reportStructure), `coverageSummary`, `insights`, `strengthProfileSummaryLLM`, `profileByDimensions`, `strengthProfileSummaryHybrid`, `careerClusterAlignment`, `_meta` (generatedAt, answerCount).
- **Invariants:** Report is built from getReportTemplate() plus session and assessment data. If cached report exists for same sessionId and answerCount, return cached; otherwise run synthesis (profile LLM, hybrid, recommendations) and cache. careerClusterAlignment has shape `{ recommended?, directionsToAvoid? }` when recommendations are enabled.
- **Internal vs external:** Template section keys are defined in reportStructure; report payload uses both template keys and additional keys (e.g. strengthProfileSummaryLLM) for LLM output. UI and any consumer depend on the presence and shape of these top-level keys.
- **Stability:** Evolving. Adding new optional report sections is allowed. Changing or removing existing keys is breaking for UI.

---

### Report structure (`src/lib/reportStructure.js`)

- **Purpose:** Single source of truth for report section keys aligned with product description (Strength Profile, Career Clusters, etc.).
- **Public interface:** `REPORT_SECTIONS` (array of section key strings); `getReportTemplate()` → object with each section key mapped to null.
- **Invariants:** Report template is used as the base object for the report; actual report payload includes extra keys (e.g. strengthProfileSummaryLLM) not in REPORT_SECTIONS. REPORT_SECTIONS is the canonical list of “expected” sections; implementation may add synthetic keys for LLM output.
- **Internal vs external:** Adding or renaming REPORT_SECTIONS affects reportService and any consumer that iterates sections. UI currently reads specific keys (e.g. strengthProfileSummaryLLM, careerClusterAlignment), not REPORT_SECTIONS.
- **Stability:** Evolving. Consistency between REPORT_SECTIONS and actual report shape is not fully enforced in code; document any divergence.

---

### Assessment model (`src/data/assessmentModel.js`)

- **Purpose:** Load and expose the dimension model (aptitudes, traits, values, skills) from JSON in src/data. Provide flat list and by-type/by-id lookups.
- **Public interface:** `load()` → cached object with `aptitudes`, `traits`, `values`, `skills`, and `*ById` Maps; `getAllDimensions()` → flat list with `dimensionType`, `dimensionId`, and item fields; `getDimension(dimensionType, dimensionId)` → item or null.
- **Invariants:** Data files are in src/data (aptitudes.json, traits.json, values.json, skills.json). Each item has at least id, name; may have description, how_measured_or_observed, question_hints. dimensionType is singular (aptitude, trait, value, skill); coverage keys elsewhere use plurals (aptitudes, traits, values, skills).
- **Internal vs external:** File paths are fixed relative to the module. Callers must not assume file names or location. Adding new fields to JSON is backward compatible.
- **Stability:** Stable. Changing dimension ids or types breaks coverage and report synthesis.

---

### Ollama client (`src/lib/ollamaClient.js`)

- **Purpose:** Low-level HTTP client for Ollama chat API (local or cloud). Uses config/llm for URL, key, model, and generation options.
- **Public interface:** `chat(messages, options)` → Promise of `{ content, done }`; `isAvailable()` → Promise of boolean; `config` (enabled, baseUrl, model, provider).
- **Invariants:** Only used when provider is ollama or ollama_cloud and config.enabled. Messages are array of { role, content }. No streaming in current implementation (stream: false).
- **Internal vs external:** Request/response logging and error handling are internal. Callers depend on content string and config.enabled.
- **Stability:** Stable. API path and body shape are tied to Ollama; changing provider would require a different client or adapter.

---

### Ollama interview (`src/lib/ollamaInterview.js`)

- **Purpose:** Build prompts and call Ollama to generate one scenario-based interview question that probes a set of dimensions. Optionally assess answers and get next question (legacy path). Validate and normalize LLM response for nextQuestion shape.
- **Public interface:** `generateScenarioQuestion(dimensionSet, askedQuestionTitles, answers, preSurveyProfile)` → `{ assessmentSummary?, nextQuestion }`. nextQuestion has title, optional description, type, options; id is not returned (assigned by assessmentService). Optional: `assessAndGetNextQuestion(answers, lastAnswerText, preSurveyProfile)` for legacy flow.
- **Invariants:** nextQuestion type is single_choice or multi_choice; options are non-empty with text and value; validation may strip invalid id. Pre-survey profile is used to build tailoring block (audience, style, tone) appended to system prompt. Personality clusters loaded from src/data/personality_clusters.json.
- **Internal vs external:** Prompt text and validation rules are internal. assessmentService depends on dimensionSet shape and return shape. Changing question schema breaks UI.
- **Stability:** Evolving. Validation and prompt content may be tuned; contract with assessmentService must be preserved.

---

### Report synthesis (`src/lib/reportSynthesis.js`)

- **Purpose:** Generate report sections using LLM: profile summary from full insights, hybrid summary from coverage, profession recommendations from profile and AI relevance ranking.
- **Public interface:** `generateProfileSummaryLLM(insights, preSurveyProfile)` → string or null; `generateProfileSummaryHybrid(coverage, model)` → `{ profileByDimensions, strengthProfileSummaryHybrid }`; `generateProfessionRecommendations(profileSummary, profileByDimensions, preSurveyProfile)` → `{ recommended, directionsToAvoid }` or null. Helpers: `getExploredDimensionsFromCoverage`, `loadAiRelevanceRanking`, `getRankingEntriesForExploredDimensions`.
- **Invariants:** All functions may return null or empty when LLM is disabled or prompt not set. profileByDimensions has keys aptitudes, traits, values, skills (arrays of { id, name }). recommended/directionsToAvoid are normalized (fit: high|medium|low; capped counts). AI relevance ranking is in src/data/ai_relevance_ranking.json; trait_id in ranking refers to dimension id in assessment model.
- **Internal vs external:** Prompt construction and normalization are internal. reportService depends on return shapes. Changing explored-dimension or recommendation shape breaks report and UI.
- **Stability:** Evolving. Adding optional fields is allowed; changing existing shapes is breaking.

---

## bft-ui

### Survey API (`src/services/surveyApi.js`)

- **Purpose:** HTTP client for session, assessment, and report. Base URL from app config (empty string when using dev proxy).
- **Public interface:** `createSession(preSurveyProfile?)` → session; `submitAnswers(sessionId, answers)` → response; `getNextQuestion(sessionId)` → next result; `getAssessment(sessionId)` → assessment; `getReport(sessionId)` → report. All return Promises; errors carry status and optional body.
- **Invariants:** Base URL is appConfig.apiBaseUrl ?? ''; in dev with proxy, empty base yields relative /api. Request body is JSON; errors are thrown with message and status.
- **Internal vs external:** Base URL and path construction are internal. Pages and components depend on function names and response shapes matching API.
- **Stability:** Evolving. Must stay in sync with API routes and response shapes.

---

### Pre-survey wizard and cluster profile

- **Purpose:** Collect optional demographics and style/cluster answers; compute cluster profile; pass profile to main survey on completion. Persist progress in localStorage.
- **Public interface:** PreSurveyWizard: `onComplete(profile)` with profile from `computeClusterProfile(questions, answers)`. Profile: `weights`, `dominant`, `secondaryTone`, `demographics`. Pre-survey questions from `src/data/pre_survey_questions.json`; structure includes intro_text, questions with id, type, options, scenario_clusters.
- **Invariants:** Pre-survey has 5 questions (Q1–Q5). Cluster weights from Q3 and Q4 only (Q4 weighted 2×); Q5 sets secondaryTone; Q1/Q2 set demographics. Tone is always default (friendly but straightforward; no tone question). No scenario-settings question. Pre-survey state is not sent to API until user starts main survey (session create with preSurveyProfile).
- **Internal vs external:** Question set and weighting rules are in UI data and clusterProfile util. API expects preSurveyProfile on session create; shape must match what report and LLM expect (dominant, secondaryTone, demographics).
- **Stability:** Evolving. Changing question ids or profile shape requires coordination with API and src/data (personality_clusters).

---

### Main survey and results flow

- **Purpose:** Create session (with optional cluster profile), loop: get next question → submit answer → get next until completed; then navigate to results. Results and Recommendations pages fetch report by sessionId (from location state or sessionStorage).
- **Public interface:** MainSurvey receives `clusterProfile`; creates session once; displays nextQuestion (id, title, type, options, maxSelections); submits answers as `[{ questionId, value }]`. On completion, stores sessionId in sessionStorage (RESULTS_SESSION_KEY) and navigates to /results. ResultsPage and RecommendationsPage resolve sessionId from location.state or sessionStorage, then getReport(sessionId).
- **Invariants:** Session id is preserved for the whole main survey and for results/recommendations. Answer value: single value for single_choice, array for multi_choice; UI normalizes to API shape (questionId, value). Progress (questionsAsked, percentComplete, etc.) is displayed from getNextQuestion response.
- **Internal vs external:** RESULTS_SESSION_KEY and session resolution logic are duplicated in ResultsPage and RecommendationsPage. Contract with API: sessionId in path; answers and report shapes as defined in API.
- **Stability:** Evolving. Session handoff and storage strategy may change; sessionId must remain the single key for report access.

---

### Config and env (UI)

- **Purpose:** Expose API base URL and app name to the app. In development, require VITE_API_PORT or API_PORT for proxy. In production build, require VITE_API_BASE_URL and VITE_APP_NAME.
- **Public interface:** `appConfig.apiBaseUrl`, `appConfig.appName` (from env.js). vite.config.js validates env and configures proxy in dev.
- **Invariants:** No code defaults for production vars; build fails if missing. In dev, empty apiBaseUrl is acceptable when proxy is used (relative /api).
- **Stability:** Stable. New optional VITE_* vars are allowed; removing required vars is breaking for build.
