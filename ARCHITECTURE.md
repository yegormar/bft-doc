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
│  │ Wizard      │──▶│ (session +       │──▶│ (report by sessionId)           │ │
│  │ (local only)│  │  assessment/next)│  │                                  │ │
│  └─────────────┘  └──────────────────┘  └─────────────────────────────────┘ │
│         │                     │                          │                   │
│         │ clusterProfile     │ HTTP /api/*               │ getReport         │
│         ▼                     ▼                          ▼                   │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ surveyApi (createSession, submitAnswers, getNextQuestion, getReport)  │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                                        │
                          (dev: Vite proxy /api → API; prod: VITE_API_BASE_URL)
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  bft-api (Express)                                                            │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │ Routes: /api/sessions | /api/sessions/:id/assessment | .../report       │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                                        │                                      │
│  ┌────────────────────────────────────┼────────────────────────────────────┐ │
│  │ Controllers (sessions, assessments, reports)                            │ │
│  └────────────────────────────────────┼────────────────────────────────────┘ │
│                                        ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │ Services: sessionService (in-memory Map)                                 │ │
│  │           assessmentService (answers, interview state, coverage, LLM)   │ │
│  │           reportService (template + synthesis, cache by answerCount)   │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│         │                    │                    │                           │
│         ▼                    ▼                    ▼                           │
│  ┌──────────────┐  ┌─────────────────┐  ┌──────────────────────────────┐   │
│  │ src/data     │  │ lib/ollamaClient │  │ lib/reportSynthesis           │   │
│  │ assessment   │  │ lib/ollama       │  │ lib/reportStructure            │   │
│  │ Model (JSON) │  │ Interview       │  │ (profile, hybrid, recommend.)  │   │
│  └──────────────┘  └─────────────────┘  └──────────────────────────────┘   │
│         │                    │                    │                           │
│         └────────────────────┴────────────────────┘                           │
│                                        │                                      │
│  ┌────────────────────────────────────▼────────────────────────────────────┐ │
│  │ config/index.js, config/llm.js (env-only; fail-fast on invalid/missing) │ │
│  │ conf/*.txt (prompt files), src/data/personality_clusters.json           │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Module boundaries and responsibilities

### bft-api

| Layer      | Location           | Responsibility |
|-----------|--------------------|----------------|
| Entry     | `server.js`, `app.js` | Create Express app, load config, mount `/api` routes, CORS, JSON, request logging, 404, error handler. |
| Config    | `config/index.js`, `config/llm.js`, `config/ollama.js` | Server and LLM configuration from environment only; process exits on invalid or missing required vars. No defaults in code for required values. |
| Routes    | `src/routes/*`     | Mount session, assessment, and report endpoints under `/api`. |
| Controllers | `src/controllers/*` | Parse request, validate session exists, call single service, send JSON response. No business logic. |
| Services  | `src/services/*`   | Business logic: sessions (CRUD, in-memory), assessment (answers, coverage, next-question generation, progress), report (template, LLM synthesis, cache). |
| Lib       | `src/lib/*`       | Ollama chat client; interview prompts and scenario-question generation; report profile/hybrid/recommendations synthesis; report section keys. |
| Data      | `src/data/*`      | Assessment model (aptitudes, traits, values, skills, ai_relevance_ranking, personality_clusters) and loader; JSON files only. |
| Middleware | `src/middleware/*` | Request logging, 404, global error handler. |
| Conf      | `conf/*`          | LLM prompt files (paths from env). Not application config. |

### bft-ui

| Layer     | Location              | Responsibility |
|----------|------------------------|----------------|
| Entry    | `main.jsx`            | React root, ChakraProvider, BrowserRouter, theme. |
| App      | `App.jsx`              | Layout, React Router routes (/, /discovery, /results, /recommendations, /about, 404). |
| Pages    | `src/components/pages/*` | Top-level route components: Home, Discovery, Results, Recommendations, About, NotFound. |
| Discovery | `PreSurveyWizard`, `MainSurvey`, `PreSurveyQuestion` | Pre-survey (local state + localStorage), then main survey (session + assessment/next loop). Pre-survey outcome (cluster profile) is sent to API on session create. |
| Layout   | `Layout`, `Header`, `PageHero` | Shell and page headers. |
| Services | `surveyApi.js`        | HTTP client for sessions, assessment answers/next, report. Base URL from env (empty in dev for proxy). |
| Config   | `src/config/env.js`, `vite.config.js` | Env: VITE_API_BASE_URL, VITE_APP_NAME (prod); VITE_API_PORT/API_PORT (dev for proxy). Build fails if required env missing. |
| Data     | `src/data/*`          | Pre-survey questions JSON, app config. |
| Utils    | `format`, `clusterProfile`, `preSurveyStorage` | Formatting, cluster profile from pre-survey answers, localStorage persist/restore for pre-survey. |

---

## Explicit layering rules

1. **Request flow (API):** HTTP → Routes → Controllers → Services. Controllers do not call lib or data directly; only services do.
2. **Services** may use: other services (e.g. reportService uses sessionService, assessmentService), lib, and data. They own in-memory state (Maps) for sessions, answers, interview state, report cache.
3. **Lib** may use: config (e.g. llm), data (e.g. assessmentModel), and other lib (e.g. ollamaClient). Lib does not depend on routes, controllers, or services.
4. **Config** is loaded at startup; it does not depend on routes, controllers, services, or lib (only on env and filesystem for prompt file paths).
5. **UI:** Pages and Discovery components call `surveyApi` for all server communication. No direct fetch to arbitrary URLs; base URL is from env. Pre-survey data stays in UI (localStorage + state) until session create.
6. **Session identity:** Session id is a ULID. UI may generate one and send it in `POST /api/sessions` body as `id`; API accepts it if valid ULID, otherwise generates a new one. All subsequent calls use `sessionId` in the path.

---

## Forbidden dependencies and anti-patterns

- **API:** Routes or controllers must not require or call `src/data` or `src/lib` directly; only services and lib may. Config must not depend on application code. No default values in code for required configuration (see workspace rule); optional env (e.g. interview limits) may have documented fallbacks.
- **UI:** No backend logic (coverage, report synthesis) in the UI; all such logic lives in the API. No hardcoded API base URL; use env. Pre-survey flow must not call the session/assessment API until the user proceeds to main survey.
- **Cross-project:** bft-ui and bft-api are separate deployables; there is no shared npm package. Contract between them is HTTP and the shape of request/response bodies (see CONTRACTS.md).
- **State:** Session, assessment, and report state are in-memory in the API. Do not assume persistence across process restarts or multiple instances.

---

## Design principles

1. **Config and env:** Required configuration is explicit and validated at startup; the process exits with a clear message if something is missing or invalid. Defaults belong in `.env.example` and documentation, not in application code.
2. **Single responsibility per layer:** Controllers handle HTTP; services hold business logic and state; lib holds LLM and report structure; data holds the assessment model.
3. **LLM behind a single client:** All Ollama/LLM calls go through `ollamaClient`; prompts are loaded from files under `conf/` via config paths. This keeps prompt content out of code and allows provider/model changes via config.
4. **Interview driven by coverage:** The main interview is dimension-coverage driven (aptitudes, traits, values, skills). Next-question selection minimizes under-covered dimensions; LLM generates scenario questions that probe multiple dimensions. Static fallback questions exist when LLM is disabled or fails.
5. **Report as synthesis:** The report is assembled from a fixed template (reportStructure), assessment data (coverage, insights), and optional LLM-generated sections (profile summary, hybrid summary, career recommendations). Cache is keyed by answer count so that new answers invalidate the cached report.
6. **Pre-survey for tailoring only:** Pre-survey results (cluster profile, demographics) are used to tailor LLM tone and scenario style and to personalize the report. They are not stored as first-class entities in the API; they are part of the session payload and passed into prompts.
