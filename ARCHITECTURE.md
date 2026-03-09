# Architecture

## Overall system purpose

**Built for Tomorrow** is an AI-powered guided interview platform for students and young adults. It helps users discover strengths (aptitudes, traits, values, skills), assess AI-era relevance of those dimensions, and receive a structured report with strength profiles and career-direction recommendations. The system does not predict a single “correct” path; it provides multi-path, strength-first, strategy-oriented guidance.

---

## High-level component diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                    User                                      │
└─────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  bft-ui (React SPA)                                                          │
│  ┌─────────────┐  ┌──────────────────┐  ┌─────────────────────────────────┐ │
│  │ PreSurvey   │  │ MainSurvey       │  │ Results / Recommendations       │ │
│  │ Wizard      │──▶│ (session +       │──▶│ (report by sessionId; core or     │ │
│  │ (local only)│  │  assessment/next)│  │  full with LLM)                    │ │
│  └─────────────┘  └──────────────────┘  └─────────────────────────────────┘ │
│         │                     │                          │                   │
│         │ clusterProfile     │ HTTP /api/* (cookie       │ getReport(include)│
│         ▼                     │  bft_uid for store)      ▼                   │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ surveyApi (createSession, submitAnswers, replaceAnswers, getNextQuestion,│   │
│  │           getReport, getOccupationsBySkillIds, submitFeedback)         │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                                        │
                          (dev: Vite proxy /api → API; prod: VITE_API_BASE_URL)
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  bft-api (Express)                                                            │
│  │ Middleware: bftUserId (cookie), requestLogger. Routes: /api/sessions,     │ │
│  │   sessions/:id/assessment, sessions/:id/report, /occupations, /feedback  │ │
│  │   (+ /api/admin when BFT_ADMIN_ENABLED=1)                                 │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                                        │                                      │
│  ┌────────────────────────────────────┼────────────────────────────────────┐ │
│  │ Controllers: sessions, assessments, reports, occupations, feedback        │ │
│  └────────────────────────────────────┼────────────────────────────────────┘ │
│                                        ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │ Services: sessionService (in-memory Map)                                 │ │
│  │           assessmentService (answers, interview state, coverage;         │ │
│  │             triangles mode = fixed set from dimension_triangles.json;    │ │
│  │             scenarios mode = questionGeneration + pregen + store)        │ │
│  │           reportService (template, core vs full report, cache by         │ │
│  │             answerCount; getReport(sessionId, { includeLlm }))            │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│         │                    │                    │                           │
│         ▼                    ▼                    ▼                           │
│  ┌──────────────┐  ┌─────────────────┐  ┌──────────────────────────────┐   │
│  │ src/data     │  │ questionGeneration│  │ lib/reportSynthesis            │   │
│  │ assessment   │  │ (FIFO queue,     │  │ lib/reportStructure            │   │
│  │ Model +      │  │  LLM+timeout,     │  │ lib/skillRecommendation         │   │
│  │ dimension_   │  │  store fallback); │  │ (profile, hybrid, skills)       │   │
│  │ triangles    │  │ lib/ollamaClient │  └──────────────────────────────┘   │
│  └──────────────┘  │ lib/ollamaInterview│  lib/occupationService, NOC data    │
│         │          └─────────────────┘  └──────────────────────────────┘   │
│         └────────────────────┴────────────────────┘                           │
│                                        │                                      │
│  ┌────────────────────────────────────▼────────────────────────────────────┐ │
│  │ config/index.js, config/llm.js, config/assessment.js,                     │ │
│  │ config/questionGeneration.js (scenarios only); config/ollama.js (legacy)  │ │
│  │ Startup: runStartupChecks (files/dirs, LLM connectivity when enabled)    │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Module boundaries and responsibilities

### bft-api

| Layer      | Location           | Responsibility |
|-----------|--------------------|----------------|
| Entry     | `server.js`, `app.js` | Create Express app, load config, mount `/api` routes, CORS, JSON, request logging, 404, error handler. |
| Config    | `config/index.js`, `config/llm.js`, `config/assessment.js`, `config/questionGeneration.js` | Server, LLM, assessment mode (triangles vs scenarios), and question-generation config from environment; process exits on invalid or missing required vars. `config/ollama.js` exists for backward compatibility (re-exports llm). |
| Routes    | `src/routes/*`     | Mount sessions, assessment, report, occupations, feedback under `/api`; optional admin when BFT_ADMIN_ENABLED=1. |
| Controllers | `src/controllers/*` | Parse request, validate session exists, call single service, send JSON response. No business logic. |
| Services  | `src/services/*`   | Business logic: sessionService (CRUD, in-memory); assessmentService (answers, coverage, next-question; triangles = fixed set from dimension_triangles.json, scenarios = questionGeneration + pregen + store); reportService (template, core vs full with includeLlm, cache by answerCount); questionStore (file store, used-set per bftUserId). |
| Lib       | `src/lib/*`       | ollamaClient; ollamaInterview (scenario question generation); reportSynthesis, reportStructure, skillRecommendation; occupationService (NOC, skills match); startupChecks, llmCheckup; bftUserId middleware. |
| Data      | `src/data/*`      | Assessment model (dimension_aptitudes, dimension_traits, dimension_values, skills, ai_relevance_ranking, personality_clusters, scenarioBatches), dimension_triangles.json for triangle mode; NOC data for occupations. |
| Middleware | `src/middleware/*` | bftUserId (cookie), requestLogger, notFound, errorHandler. |
| Conf      | `conf/*`          | LLM prompt files (paths from env). Not application config. |

### bft-ui

| Layer     | Location              | Responsibility |
|----------|------------------------|----------------|
| Entry    | `main.jsx`            | React root, ChakraProvider, BrowserRouter, theme. |
| App      | `App.jsx`              | Layout, React Router routes (/, /discovery, /results, /recommendations, /skills, /feedback, /about, 404). |
| Pages    | `src/components/pages/*` | Top-level route components: Home, Discovery, Results (profile, traits-values, answers), Recommendations, Skills, Feedback, About, NotFound. |
| Discovery | `PreSurveyWizard`, `MainSurvey`, triangle/single-choice questions | Pre-survey (local state + localStorage), then main survey (session + assessment/next; triangle or scenario questions). Pre-survey outcome sent to API on session create. |
| Layout   | `Layout`, `Header`, `PageHero` | Shell and page headers. |
| Services | `surveyApi.js`        | HTTP client for sessions, assessment (submitAnswers, replaceAnswers, getNextQuestion, getAssessment), report (getReport with includeLlm), occupations, feedback. Base URL from env (empty in dev for proxy). |
| Config   | `src/config/env.js`, `vite.config.js` | Env: VITE_API_BASE_URL, VITE_APP_NAME (prod); build fails if required env missing. |
| Data     | `src/data/*`          | Pre-survey questions JSON, app config. |
| Utils    | `format`, `clusterProfile`, `preSurveyStorage` | Formatting, cluster profile from pre-survey answers, localStorage persist/restore for pre-survey. |

---

## Explicit layering rules

1. **Request flow (API):** HTTP → bftUserId (when under /api) → Routes → Controllers → Services. Controllers do not call lib or data directly; only services do.
2. **Services** may use: other services (e.g. reportService uses sessionService, assessmentService), lib, and data. They own in-memory state (Maps) for sessions, answers, interview state, pregen queues, report cache.
3. **Lib** may use: config (llm, assessment, questionGeneration), data (assessmentModel), and other lib. Lib does not depend on routes, controllers, or services.
4. **Config** is loaded at startup (config/index, llm, assessment are required; questionGeneration is loaded only when assessment mode is scenarios). Config does not depend on application code (only env and filesystem for prompt/file paths).
5. **UI:** Pages and Discovery components call `surveyApi` for all server communication. No direct fetch to arbitrary URLs; base URL is from env. Pre-survey data stays in UI until session create. Cookie bft_uid is set by API for question-store "used" tracking.
6. **Session identity:** Session id is a ULID. UI may send `id` in `POST /api/sessions`; API uses it if valid ULID, else generates one. All subsequent calls use sessionId in the path. Report can be requested with core data only (no LLM) or full (includeLlm: true) for profile/recommendations pages.
7. **Assessment modes:** BFT_ASSESSMENT_MODE=triangles uses a fixed question set from dimension_triangles.json; no LLM or question store for next question. BFT_ASSESSMENT_MODE=scenarios uses questionGeneration (FIFO queue, LLM with timeout, store fallback) and optional background pregen.

---

## Forbidden dependencies and anti-patterns

- **API:** Routes or controllers must not require or call `src/data` or `src/lib` directly; only services and lib may. Config must not depend on application code. No default values in code for required configuration (see workspace rule); optional env (e.g. interview limits) may have documented fallbacks.
- **UI:** No backend logic (coverage, scoring, report synthesis) in the UI; all such logic lives in the API. No hardcoded API base URL; use env. Pre-survey flow must not call the session/assessment API until the user proceeds to main survey.
- **Cross-project:** bft-ui and bft-api are separate deployables; no shared npm package. Contract is HTTP and request/response shapes (see CONTRACTS.md).
- **State:** Session, assessment, and report state are in-memory in the API. Do not assume persistence across process restarts or multiple instances.
- **Report:** replaceAnswers invalidates report cache (reportService.invalidateReportCache); only answer-count change is used as cache key. Edits after completion use replaceAnswers with same length; no new question generation.

---

## Design principles

1. **Config and env:** Required configuration is explicit and validated at startup; the process exits with a clear message if something is missing or invalid. Defaults belong in `.env.example` and documentation, not in application code.
2. **Single responsibility per layer:** Controllers handle HTTP; services hold business logic and state; lib holds LLM and report structure; data holds the assessment model.
3. **LLM behind a single client:** All Ollama/LLM calls go through `ollamaClient`; prompts are loaded from files under `conf/` via config paths. This keeps prompt content out of code and allows provider/model changes via config.
4. **Interview modes:** Triangles mode: fixed set of questions from dimension_triangles.json, one question per triangle; scores derived from barycentric coordinates (scale 1-5). Scenarios mode: coverage-driven; next question minimizes under-covered dimensions; questionGeneration (LLM with timeout, then store fallback), optional background pregen; single_choice, multi_choice, rank types with dimensionScores on options.
5. **Report as synthesis:** Report is assembled from a fixed template (reportStructure), assessment data (coverage, insights, dimensionScores), and optional LLM sections when includeLlm is true (profile summary, hybrid summary). Core report (no LLM) is used for traits-values and skills pages; full report for profile and recommendations. Cache is keyed by session and answer count.
6. **Pre-survey for tailoring only:** Pre-survey results (cluster profile, demographics) are sent on session create and passed into LLM prompts. They are not first-class entities; they are part of the session payload.
7. **Startup checks:** Before listening, runStartupChecks ensures required files/dirs (questions store, feedback file, data files, scenario step prompt files) exist and are writable where needed, and runs LLM connectivity check when LLM is enabled.
