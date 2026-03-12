# Multi-Language Support: What Is Needed

This document outlines what the application needs to support different languages in three areas: UI, LLM-generated answers, and data loaded from JSON files.

---

## 1. UI Changes

### Current state

- **Header** (`bft-ui/src/components/Layout/Header.jsx`): A language dropdown exists with `LOCALES = [{ value: 'en', label: 'English' }, { value: 'fr', label: 'Français' }]`. The selected value is stored in `locale` state and `localStorage` key `locale`, but nothing else in the app reads it.
- **i18n** (`bft-ui/src/i18n/`): Minimal setup: `i18n.js` exports `defaultLocale` and `supportedLocales`; `en.json` has a few keys (`common.home`, `home.welcome`, etc.). No integration with components: no `useTranslation`, no `t()`, so UI copy is not coming from translation files.
- **Copy**: Most text is hardcoded in components (e.g. HomePage reframing cards, trust items, nav labels "Discovery", "Results", "Feedback", "About", button "Start discovery", page titles, error messages).

### What is needed

1. **Use a real i18n library**  
   Integrate e.g. `react-i18next` (or similar) so components can call `t('key')` and the active locale drives which JSON is used.

2. **Locale available app-wide**  
   Either:
   - Provide locale via React context (or the i18n provider) so the Header and all pages use the same value, or
   - Keep reading from `localStorage` in a single place and pass locale into the i18n library so it loads the right resource bundle.

3. **Translation JSON per language**  
   - Add e.g. `bft-ui/src/i18n/fr.json` (and any other locales).  
   - Structure keys to match the current UI (e.g. `home.heroTitle`, `home.heroTagline`, `nav.discovery`, `nav.results`, etc.).

4. **Replace hardcoded strings with `t()`**  
   In every page and layout component, replace literal copy with `t('namespace.key')` and put that copy into the locale JSON files. Areas to cover:
   - **Layout**: Header (nav links, "Select language", color mode aria-label), footer if any.
   - **Pages**: HomePage (hero, reframing cards, trust items, CTAs), AboutPage, DiscoveryPage, ResultsPage and all Results* sub-pages, RecommendationsPage, SkillsPage, FeedbackPage, NotFoundPage.
   - **Shared**: Buttons, form labels, validation messages, empty states, loading/error messages in `surveyApi` or components.

5. **Pre-survey and other UI-loaded JSON**  
   `bft-ui/src/data/pre_survey_questions.json` holds user-facing text (intro, question titles, options). For multi-language:
   - Either one JSON per locale (e.g. `pre_survey_questions.en.json`, `pre_survey_questions.fr.json`) and load by locale, or
   - One file with nested keys by locale and pick the right branch when loading. Same pattern applies to any other UI-only JSON (e.g. `appConfig.json` if it contains user-facing strings).

6. **RTL (optional)**  
   If you add a right-to-left language, set `dir` and possibly layout/CSS from the active locale.

7. **E2E and tests**  
   Update tests that assert on exact copy (e.g. "Built for Tomorrow", "about page") to use translation keys or locale-aware expectations.

---

## 2. LLM Answers

### Current state

- **Report profile summary** (`bft-api/src/lib/reportSynthesis.js`): One LLM call using a system prompt from `conf/report_profile_system_prompt.txt` and optional context from `conf/report_ai_context.txt`. Output is HTML (personality, study paths, careers). No language or locale is passed; prompts and model output are effectively English-only.
- **Career paths** (`bft-api/src/lib/careerPathsSynthesis.js`): Two-step LLM flow using `conf/career_paths_step1.txt`, `conf/career_paths_step2.txt`, and `CAREERS_LLM_AI_CONTEXT_FILE`. Output is structured (study, initial job, ultimate job, rationale). No locale.
- **Interview question generation** (`bft-api/src/services/questionGeneration/`, `ollamaInterview.js`): Scenario questions are generated from dimension data and prompts (e.g. scenario steps in `conf/`). Dimension names and prompts are English; no locale parameter.

All report and career-path APIs are called without a language parameter (e.g. `getReport(sessionId, { includeLlm: true })` in the UI; no `Accept-Language` or `locale` query/body).

### What is needed

1. **Pass locale/language into the API**  
   - **Option A**: UI sends preferred language on every request that can return LLM content (e.g. `GET /sessions/:id/report?locale=fr`, or `Accept-Language: fr`).  
   - **Option B**: Store preferred language on the session (e.g. at discovery start or in session PATCH) and have the API read it when generating report/career paths.  
   Decide one place of truth (request vs session) and use it consistently.

2. **Report profile summary**  
   - **Prompt**: Either (a) use a different prompt file per language (e.g. `report_profile_system_prompt.fr.txt`) selected by locale, or (b) use a single prompt that includes an instruction like "Respond in the same language as the user interface: French" (and pass the locale name/language from the request or session).  
   - **Context**: If `report_ai_context.txt` contains language-specific content, either per-locale files or a single file with language-specific sections and inject the right part.  
   - **Output**: The model should respond in the requested language; no extra parsing if the instruction is clear.

3. **Career paths synthesis**  
   - Same idea: per-locale prompt files (e.g. `career_paths_step1.fr.txt`) or one prompt with "Output in French" (or the requested language).  
   - Step 2 (formatting) can stay language-agnostic if it only parses structure; step 1 content (study, jobs, rationale) should be in the target language.

4. **Interview question generation**  
   - Dimension names and scenario text sent to the LLM come from API JSON (`dimension_traits.json`, etc.). If those are localized (see section 3), the same payload can be used with a locale so the LLM sees French (or other) dimension names and is instructed to generate the next question in that language.  
   - Scenario step prompts in `conf/` would need to be per-language or include a "generate in language X" instruction and receive the locale.

5. **Caching**  
   Report cache is currently keyed by `sessionId` and `answerCount`. If the same session can request report in different languages, cache key should include `locale` (or language) so you do not serve English summary for a French request.

6. **Config**  
   If you use per-locale prompt files, config (or env) must define paths per locale (e.g. `LLM_REPORT_PROFILE_PROMPT_FILE` vs `LLM_REPORT_PROFILE_PROMPT_FILE_FR`), or a single convention like `report_profile_system_prompt.{locale}.txt`. Per-workspace rules: no defaults in code; document every variable in `.env.example`.

---

## 3. Data Extracted from JSON Files

### Current state

- **API data (bft-api)**  
  All content is English-only:
  - **Dimensions**: `dimension_aptitudes.json`, `dimension_traits.json`, `dimension_values.json` – each item has `id`, `name`, `short_label`, `description`, and other text fields used in report payload and UI.
  - **Skills**: `skills.json` – `id`, `name`, `short_label`, `description`, and other strings.
  - **Triangles**: `dimension_triangles.json`, `dimension_triangles_aptitudes.json` – labels, option text, `answer_interpretations` (e.g. "balanced", "dominant_a").
  - **NOC**: `noc-2021-enriched.json`, `noc-2021-major-groups.json` – occupation titles, category labels, descriptions used for recommendations and occupation lists.
  - **Other**: `personality_clusters.json`, `scenarioBatches.json`, `ai_skills_ranking_model.json`, `ai_relevance_ranking.json` – contain labels/descriptions used in logic or UI.

- **UI data (bft-ui)**  
  - `pre_survey_questions.json`: intro, question titles, options (already mentioned in section 1).  
  - `appConfig.json`: app name, version, etc.; localize if user-facing.

These JSONs are loaded without a locale; the API and UI assume one language.

### What is needed

1. **Strategy: per-locale files vs nested keys**  
   - **Per-locale files**: e.g. `dimension_traits.en.json`, `dimension_traits.fr.json`. Load the file that matches the request/session locale. Pros: simple loading, no huge single file. Cons: many files; must keep structure and IDs in sync across locales.  
   - **Nested keys**: one file per entity type with a top-level key per locale, e.g. `{ "en": { "traits": [...] }, "fr": { "traits": [...] } }`. Load once and pick `data[locale]`. Pros: one file per type. Cons: file size; merge/sync across locales is harder.  
   Choose one strategy and use it consistently for all translatable JSON.

2. **API dimension and skill data**  
   - For each dimension/skill JSON, provide translated `name`, `short_label`, `description`, and any other user-visible strings. Keep `id` and structural fields (e.g. `structural_scores`, `related_skill_clusters`) locale-independent.  
   - **Report payload**: When building the session payload for the LLM (and for the UI), include dimension/skill names and descriptions in the requested locale so the LLM and UI show the right language.  
   - **Assessment model** (`assessmentModel.js`): Load the correct locale’s data (or the correct branch) based on config or request. If the API is stateless, the locale must come from the request (or session) every time.

3. **Triangle and interpretation text**  
   - Labels and `answer_interpretations` in triangle JSONs are shown in the UI and can be sent to the LLM. Either duplicate the JSON per locale or add a layer (e.g. `interpretations: { en: {...}, fr: {...} }`) and select by locale.

4. **NOC and occupations**  
   - NOC data is large and often standardized in one language. Options: (a) provide a separate NOC dataset per language (e.g. official French NOC if available), or (b) keep NOC in one language and only localize your own labels (e.g. major group labels in `noc-2021-major-groups.json`) via per-locale files or nested keys. Occupation names and descriptions shown in the app should come from the chosen locale’s source.

5. **Personality clusters, scenario batches, AI rankings**  
   - Any field that is shown to the user or sent to the LLM (e.g. cluster labels, batch themes, dimension labels in AI model) should have a locale-aware source. Prefer the same strategy as dimensions/skills (per-locale files or nested by locale).

6. **Loading and fallback**  
   - If a locale is requested but missing for a given JSON, define a fallback (e.g. `en`) and document it. Do not hardcode fallbacks in code beyond what is in config; document in `.env.example` if a default locale is configurable.

7. **UI-only JSON**  
   - `pre_survey_questions.json` and similar: load by locale in the frontend (or request from an API that returns the right locale). Same fallback rule.

---

## Summary Table

| Area              | Current state                         | Main work |
|-------------------|----------------------------------------|-----------|
| **UI**            | Locale in Header only; no i18n usage; all copy hardcoded | Add react-i18next (or similar), locale context, translation JSON per language, replace all copy with `t()`, localize pre-survey (and other UI) JSON. |
| **LLM answers**   | No locale; prompts and output English  | Pass locale/language on report and career-path (and optionally next-question) requests; per-locale or single prompt with language instruction; key report cache by locale. |
| **JSON data**     | All API/UI JSON is English-only        | Per-locale files or nested-by-locale for dimensions, skills, triangles, NOC labels, personality/scenario/AI text; load by request/session locale; keep IDs/structure stable. |

---

## Suggested order of work

1. **UI**: Wire i18n library and locale from Header; add `fr.json` (and keys in `en.json`) for one full page (e.g. Home); replace that page’s copy with `t()`. Then do the rest of the pages and shared components; then pre-survey and other UI JSON.
2. **API report/LLM**: Add `locale` (or `Accept-Language`) to report and career-path endpoints; key cache by locale; add language instruction to prompts (or one set of French prompts) and verify output language.
3. **JSON data**: Introduce locale into assessment/report data loading (dimensions, skills, triangles); add French (or one other language) for a small set of dimensions/skills; then extend to all content and NOC/personality/scenario/AI files as needed.

This keeps a clear path: first the UI speaks two languages, then the LLM output, then the static content that feeds both.
