# Contracts

This document describes the purpose, public interfaces, invariants, and stability of each major module. Interfaces are conceptual; exact field names and types are implied by the code and API behavior.

---

## bft-api

### Config (`config/index.js`, `config/llm.js`, `config/assessment.js`, `config/questionGeneration.js`)

- **Purpose:** Provide server, LLM, assessment mode, and question-generation configuration from environment. Ensure no required value is missing or invalid at startup.
- **Public interface:** index: `port`, `nodeEnv`, `corsOrigin`, `questionsStoreDir`, `feedbackFile`. llm: `provider`, `baseUrl`, `apiKey`, `model`, `temperature`, `maxTokens`, `topP`, `numCtx`, `think`, `thinkLevel`, report prompt file paths and getters, `checkupIntervalSec`, `enabled`. assessment: `getAssessmentMode()` (triangles | scenarios), `getPregenConfig()` (queueCap, refillThreshold; 0 when triangles), `getInterviewConfig()` (minSignalPerDimension, maxQuestions). questionGeneration: `getQuestionGenConfig()` (timeoutMs, storeFirst); loaded only when assessment mode is scenarios. config/ollama.js re-exports llm for backward compatibility.
- **Invariants:** Process exits on missing/invalid required env. No code defaults for required vars. BFT_ASSESSMENT_MODE required (triangles or scenarios); when scenarios, pregen and interview env required; when triangles, pregen is 0. LLM report prompt files must exist when required. questionGeneration required only when mode=scenarios.
- **Internal vs external:** Config may add optional env. Callers must not assume defaults for required keys.
- **Stability:** Stable. Adding optional env is allowed; changing required vars or assessment modes is breaking.

---

### Session service (`src/services/sessionService.js`)

- **Purpose:** Create, read, and update sessions in memory. Session is the root entity for assessment and report.
- **Public interface:** `create(preSurveyProfile, clientId)` → session; `getById(id)` → session or null; `update(id, updates)` → session or null. Session: id, status, createdAt, updatedAt, preSurveyProfile (optional). Status: draft, in_progress, completed. GET sessions/:id/health is implemented by sessionsController.getHealth and returns assessmentService.getSessionHealth(id) (preGeneratedQuestions, questionsAsked, coverage, dimensionScores, interviewComplete, backgroundPregenRunning, etc.).
- **Invariants:** Session id is a valid ULID (26 chars, Crockford base32). If clientId is provided and valid ULID, it is used as id; otherwise a new ULID is generated. At most one session per id in the Map.
- **Internal vs external:** Storage is in-memory Map; may be replaced with a persistent store without changing the public interface. Callers must not rely on process lifetime.
- **Stability:** Evolving. Adding optional session fields is allowed; changing id format or required fields is breaking.

---

### Assessment service (`src/services/assessmentService.js`)

- **Purpose:** Store answers per session, maintain interview state (coverage, asked questions, dimension mapping, dimensionScoresAggregate), determine next question (triangles: fixed set from dimension_triangles.json; scenarios: questionGeneration + pregen + store), and expose assessment and progress.
- **Public interface:** `submitAnswers(sessionId, payload)` → array of answers; `replaceAnswers(sessionId, payload)` → array of answers (only when payload.answers.length === current length); `getAssessment(sessionId)` → assessment (sessionId, answers, assessmentSummaries, coverageSummary, insights, dimensionScores { traits, values }, askedQuestionsWithAnswers); `getNextQuestion(sessionId, bftUserId)` → Promise of `{ completed, nextQuestion, assessmentSummary?, progress }` or `{ serviceUnavailable, message, retryAfterSeconds, progress }`; `getSessionHealth(sessionId)` → health object or null. Progress: questionsAsked, coveredDimensions, totalDimensions, percentComplete. Next question: id, title, type (single_choice | multi_choice | rank | triangle), options (and for triangle: vertices, prompt, framing).
- **Invariants:** submitAnswers appends answers; replaceAnswers replaces all answers and rebuilds coverage/dimension scores (same logic as submitAnswers); replaceAnswers is only allowed when payload.answers length equals current answer count. replaceAnswers does not advance the interview or trigger question generation. Triangle mode: one question per triangle from dimension_triangles.json; answer value is barycentric { a, b, c }; scores mapped to 1-5. Scenarios: interview complete when coverage meets minSignalPerDimension or maxQuestions reached; getNextQuestion may return serviceUnavailable when LLM and store both fail. dimensionScores in getAssessment are built from dimensionScoresAggregate (and synthetic fill when maxQuestions cap reached).
- **Internal vs external:** Coverage keys and dimension types are part of the model. Callers depend on getAssessment and getNextQuestion response shape. bftUserId is used for question-store "used" tracking in scenarios mode.
- **Stability:** Evolving. Adding optional fields is allowed. Changing completion rules, question types, or dimensionScores shape is breaking.

---

### Report service (`src/services/reportService.js`)

- **Purpose:** Build the report for a session from template, assessment data, and optional LLM synthesis. Cache by session and answer count; invalidate on replaceAnswers.
- **Public interface:** `getReport(sessionId, options)` → Promise of report; options.includeLlm (boolean) controls whether to run LLM synthesis (profile summary). When includeLlm is false, returns core report only (dimensionScores, dimensionMeta, profileByDimensions from coverage, skillDevelopmentRoadmap). When includeLlm is true, runs synthesis and caches by answerCount. `invalidateReportCache(sessionId)` clears cache for that session (called by controller on replaceAnswers). Report includes sessionId, status, template keys, coverageSummary, insights, dimensionScores, dimensionMeta, profileByDimensions, profileSummary, careerClusterAlignment, skillDevelopmentRoadmap, structuralDimensionMeta, _meta.
- **Invariants:** Core report is always built from template and assessment. Cached full report is returned when sessionId and answerCount match. careerClusterAlignment is null in current implementation. Cache key is answer count only.
- **Internal vs external:** Template from reportStructure; report adds profileSummary from LLM. UI depends on dimensionScores, dimensionMeta, profileByDimensions, profileSummary, skillDevelopmentRoadmap.
- **Stability:** Evolving. Adding optional sections is allowed. Changing or removing existing keys is breaking for UI.

---

### Report structure (`src/lib/reportStructure.js`)

- **Purpose:** Single source of truth for report section keys aligned with product description (Strength Profile, Career Clusters, etc.).
- **Public interface:** `getReportTemplate()` returns an object with each section key mapped to null. Section keys include coreAdvantageAreas, careerClusterAlignment, aiResilienceAnalysis, suggestedCollegeDirections, skillDevelopmentRoadmap, scenarioPlanning, backupPathStrategy.
- **Invariants:** Template is the base for the report; report payload includes extra keys (profileSummary, profileByDimensions, dimensionScores, dimensionMeta) not in the template. UI reads specific keys, not the section list.
- **Internal vs external:** Changing section keys affects reportService. Document divergence between template and actual payload.
- **Stability:** Evolving.

---

### Assessment model (`src/data/assessmentModel.js`)

- **Purpose:** Load and expose the dimension model (aptitudes, traits, values, skills) from JSON in src/data. Provide flat list and by-type/by-id lookups. Triangle questions are not in this model; they are in dimension_triangles.json and loaded in assessmentService.
- **Public interface:** `load()` → cached object with aptitudes, traits, values, skills and aptitudesById, traitsById, valuesById, skillsById, dimensionsById; `getAllDimensions()` → flat list with dimensionType, dimensionId, and item fields; `getDimension(dimensionType, id)` → item or null.
- **Invariants:** Data files: dimension_aptitudes.json, dimension_traits.json, dimension_values.json, skills.json. Each item has at least id, name; may have description, how_measured_or_observed, question_hints, score_scale. dimensionType is singular; coverage keys use plurals (aptitudes, traits, values, skills).
- **Internal vs external:** File paths are fixed relative to the module. Callers must not assume file names.
- **Stability:** Stable. Changing dimension ids or types breaks coverage and report.

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

- **Purpose:** Generate report sections using LLM: profile summary from full session payload (one call).
- **Public interface:** `getExploredDimensionsFromCoverage(coverage, model)` → `{ aptitudes, traits, values, skills }` (arrays of { id, name }); `generateProfileSummaryFromPayload(payload)` → string or null.
- **Invariants:** generateProfileSummaryFromPayload returns null when LLM is disabled or prompt not set. profileByDimensions from getExploredDimensionsFromCoverage has keys aptitudes, traits, values, skills.
- **Internal vs external:** Prompt is loaded from config (report_profile_system_prompt.txt). reportService depends on return shapes.
- **Stability:** Evolving. Adding optional fields is allowed; changing existing shapes is breaking.

---

### Question store (`src/services/questionStore.js`)

- **Purpose:** File-based persistent store for generated questions keyed by profile; per-user "used" tracking so the same question is not reused for a user. When BFT_QUESTIONS_STORE_DIR is unset, save/list are no-ops and used tracking is in-memory only.
- **Public interface:** `getProfileKey(preSurveyProfile)` → string (hash); `computeContentHash(question)` → string; `save(storeDir, profileKey, question, dimensionSet, assessmentSummary)`; `listByProfile(storeDir, profileKey)` → array of { question, dimensionSet, assessmentSummary, createdAt, contentHash }; `getUsedSet(storeDir, bftUserId)` → Set of contentHash; `markUsed(storeDir, bftUserId, contentHash)`.
- **Invariants:** Profile key is stable for same preSurveyProfile. Content hash is from question title and option texts. Used set is persisted under storeDir/used/{bftUserId}. listByProfile returns items sorted by createdAt ascending.
- **Internal vs external:** Directory layout and file naming are internal. assessmentService and questionGeneration depend on save/list/used for store-first and fallback.
- **Stability:** Evolving. Used-set format and store layout may change with backward compatibility.

---

### Question generation (`src/services/questionGeneration/index.js`)

- **Purpose:** Single entry point for requesting one scenario question: FIFO queue, then LLM with timeout, then store fallback. Used only when BFT_ASSESSMENT_MODE=scenarios.
- **Public interface:** `requestQuestion(context)` → Promise of `{ question, dimensionSet?, assessmentSummary?, source: 'llm'|'store' }` or `{ question: null, reason: string }`. context: sessionId, bftUserId, preSurveyProfile, storeDir, desiredDimensionSet, askedQuestionTitles, answers.
- **Invariants:** When storeFirst is true, try store then LLM; when false, try LLM then store. LLM timeout from config (BFT_QUESTION_LLM_TIMEOUT_MS). dimensionSet returned with each dimension having id (same as dimensionId). Returns null when LLM disabled and store has no unused match.
- **Internal vs external:** Queue, timeout, and store selection logic are internal. assessmentService is the only caller; contract is requestQuestion in/out.
- **Stability:** Evolving. Adding context fields or return fields is allowed; changing semantics is breaking.

---

### bftUserId middleware (`src/middleware/bftUserId.js`)

- **Purpose:** Ensure req.bftUserId is set from cookie bft_uid, or generate and set a new ULID and set cookie. Used for question-store "used" tracking across sessions.
- **Public interface:** `bftUserIdMiddleware(config)` → Express middleware. Sets req.bftUserId (string, ULID). Cookie: bft_uid, httpOnly, 1 year; sameSite none + secure in production, lax otherwise.
- **Invariants:** If cookie missing or invalid (not 26-char Crockford base32), generate new ULID and set cookie. config.nodeEnv used for cookie options.
- **Internal vs external:** Cookie name and options are internal. Controllers and assessmentService use req.bftUserId when calling getNextQuestion or question store.
- **Stability:** Stable.

---

### Occupations (`src/controllers/occupationsController.js`, `src/lib/occupationService.js`)

- **Purpose:** List occupations scored by selected skill IDs (from NOC 2021 enriched data); get single occupation by NOC code.
- **Public interface (API):** GET `/occupations?skillIds=...&groupBy=category` → { groups: [ { categoryKey, categoryLabel, occupations } ] }; GET `/occupations/:nocCode` → single occupation. occupationService: list by skill IDs, optional groupByCategory; getByNocCode(nocCode).
- **Invariants:** skillIds are from assessment model (skills.json). Match score from enriched NOC mappings. When groupByCategory is true, response is grouped; otherwise flat list.
- **Internal vs external:** Data from noc-2021-enriched.json (or major groups). UI Careers/Recommendations use getOccupationsBySkillIds and getOccupationByNocCode.
- **Stability:** Evolving.

---

### Feedback (`src/routes/feedback.js`, `src/controllers/feedbackController.js`)

- **Purpose:** Append user feedback (rating, optional text) to a file (one JSON line per submission). Used by Feedback page.
- **Public interface (API):** POST `/feedback` with body { rating (1-5), improve?, good? } → 201. Writes to BFT_FEEDBACK_FILE (path from config).
- **Invariants:** rating required, 1-5. improve and good are optional strings. Each submission is one line (JSON) appended to file.
- **Internal vs external:** File path from config. No reading of feedback in app; external tooling for analysis.
- **Stability:** Stable.

---

## bft-ui

### Survey API (`src/services/surveyApi.js`)

- **Purpose:** HTTP client for session, assessment, and report. Base URL from app config (empty string when using dev proxy).
- **Public interface:** `createSession(preSurveyProfile?)` → session; `submitAnswers(sessionId, answers)` → response; `replaceAnswers(sessionId, payload)` → response (PUT, same-length answers; server rebuilds coverage/scores and invalidates report cache); `getNextQuestion(sessionId)` → next result (or serviceUnavailable); `getAssessment(sessionId)` → assessment; `getReport(sessionId, options)` → report (options.includeLlm: true for profile/recommendations pages, false/omit for core only); `getOccupationsBySkillIds(skillIds, groupByCategory)`; `getOccupationByNocCode(nocCode)`; `submitFeedback({ rating, improve?, good? })`. All return Promises; errors carry status and optional body.
- **Invariants:** Base URL is appConfig.apiBaseUrl ?? ''; in dev with proxy, empty base yields relative /api. Request body is JSON; errors are thrown with message and status.
- **Internal vs external:** Base URL and path construction are internal. Pages and components depend on function names and response shapes matching API.
- **Stability:** Evolving. Must stay in sync with API routes and response shapes.

---

### Pre-survey wizard and cluster profile

- **Purpose:** Collect optional demographics and style/cluster answers; compute cluster profile; pass profile to main survey on completion. Persist progress in localStorage.
- **Public interface:** PreSurveyWizard: `onComplete(profile)` with profile from `computeClusterProfile(questions, answers)`. Profile: `weights`, `dominant`, `secondaryTone`, `demographics`. Pre-survey questions from `src/data/pre_survey_questions.json`; structure includes intro_text, questions with id, type, options, scenario_clusters.
- **Invariants:** Pre-survey has 5 questions (Q1-Q5). Cluster weights from Q3 and Q4 only (Q4 weighted 2×); Q5 sets secondaryTone; Q1/Q2 set demographics. Tone is always default (friendly but straightforward; no tone question). No scenario-settings question. Pre-survey state is not sent to API until user starts main survey (session create with preSurveyProfile).
- **Internal vs external:** Question set and weighting rules are in UI data and clusterProfile util. API expects preSurveyProfile on session create; shape must match what report and LLM expect (dominant, secondaryTone, demographics).
- **Stability:** Evolving. Changing question ids or profile shape requires coordination with API and src/data (personality_clusters).

---

### Main survey and results flow

- **Purpose:** Create session (with optional cluster profile), loop: get next question → submit answer → get next until completed; then navigate to results. Results and Recommendations pages fetch report by sessionId (from location state or sessionStorage).
- **Public interface:** MainSurvey receives `clusterProfile`; creates session once; displays nextQuestion (id, title, type, options; type can be triangle with vertices/prompt). Submits answers as `[{ questionId, value }]` (value: string for single_choice, array for multi_choice/rank, { a, b, c } for triangle). On completion, stores sessionId in sessionStorage (RESULTS_SESSION_KEY) and navigates to /results. Results pages resolve sessionId from location.state or sessionStorage; getReport(sessionId, { includeLlm: true }) for profile/recommendations, core only for traits-values/skills. GET sessions/:id/health available for debug/pregen status.
- **Invariants:** Session id is preserved for the whole survey and results. Answer value shape matches question type. Progress from getNextQuestion response. When getNextQuestion returns serviceUnavailable, UI may show retry message with retryAfterSeconds.
- **Internal vs external:** RESULTS_SESSION_KEY and session resolution logic are duplicated in ResultsPage and RecommendationsPage. Contract with API: sessionId in path; answers and report shapes as defined in API.
- **Stability:** Evolving. Session handoff and storage strategy may change; sessionId must remain the single key for report access.

---

### Config and env (UI)

- **Purpose:** Expose API base URL and app name to the app. In development, require VITE_API_PORT or API_PORT for proxy. In production build, require VITE_API_BASE_URL and VITE_APP_NAME.
- **Public interface:** `appConfig.apiBaseUrl`, `appConfig.appName` (from env.js). vite.config.js validates env and configures proxy in dev.
- **Invariants:** No code defaults for production vars; build fails if missing. In dev, empty apiBaseUrl is acceptable when proxy is used (relative /api).
- **Stability:** Stable. New optional VITE_* vars are allowed; removing required vars is breaking for build.
