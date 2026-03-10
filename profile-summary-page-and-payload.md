# Profile Summary page: single approach from session payload

## Current implementation

### Data flow

- **Page:** `bft-ui/src/components/pages/ResultsProfilePage.jsx`
- **API:** `getReport(sessionId, { includeLlm: true })` → `GET /api/sessions/:sessionId/report?include=full`
- **Backend:** When `includeLlm` is true, `reportService.getReport()` builds the session payload via `getSessionPayloadForLlm(sessionId)`, runs one LLM call (`generateProfileSummaryFromPayload(payload)`), and exposes `report.profileSummary` (single string). System prompt: `conf/report_profile_system_prompt.txt`.

### What the page shows

- One narrative block: **"Your discovered strengths and profile"** – `report.profileSummary` (if present). The LLM is prompted to output three sections:
  1. **Your personality at a glance** – horoscope-style, insight about how their strengths show up.
  2. **Study paths that could suit you** – 3–5 study directions with short rationale.
  3. **Careers and opportunities that could benefit you most** – 3–5 career clusters with short rationale.
- The LLM may use basic HTML for structure: `<h2>`, `<h3>`, `<b>`, `<table>`, `<tr>`, `<td>`, `<th>`. The UI sanitizes and renders this HTML; other tags are stripped.
- No dimensions, no Q&A, no skills, no pretest profile on the page. If the LLM call fails or is disabled, the page shows: "No profile summary yet."

### Limitations

1. **No evidence** – User does not see what was measured (dimensions, scores) or which answers drove the summary.
2. **No fallback** – If the LLM is slow or errors, the page is empty except for the fallback message.
3. **No link to skills** – Skills that fit the profile are not shown; user must go to Skills separately.
4. **Personality cluster hidden** – Pre-survey profile (e.g. dominant, tone) is not shown, so the “personality” side of the assessment is invisible.
5. **Two separate API shapes** – Report has many keys (dimensionScores, dimensionMeta, skillDevelopmentRoadmap, etc.); the payload is a single, consistent structure designed for summary + recommendations.

---

## The session payload (JSON for LLM)

The payload from `GET /api/sessions/:sessionId/report/payload` is a single JSON object with:

- **questions_and_answers** – All questions and the user’s answer (triangle barycentric or choice).
- **dimensions** – `aptitudes`, `traits`, `values`, each with: id, name, description, how_measured_or_observed, related_skill_clusters, score_scale, **mean**, **band**, **count**.
- **skills** – id, name, description, **applicability** (score from dimension measurements).
- **personality_cluster** – pre_survey_profile + questions_and_answers_for_profile.

See **bft-doc/session-payload-for-llm.md** for the full shape.

---

## How the Profile Summary page could be improved

### 1. Use the payload as the main data source for the page

- **Add:** `getReportPayload(sessionId)` in `surveyApi.js` calling `GET .../report/payload`.
- **Option A:** Fetch both report (for LLM narratives) and payload (for structured data). Page then has narrative + dimensions + skills + personality in one place.
- **Option B:** Fetch only the payload; run a single LLM call (e.g. from backend) that receives this JSON and returns (a) summary and (b) profession recommendations. Then the page displays summary + optional “evidence” sections from the same payload.

### 2. Show “What we measured” (dimensions)

- Use **payload.dimensions** (aptitudes, traits, values).
- For each dimension show: **name**, **mean** (e.g. 3.2) and **band** (low/medium/high), optional short **description** or **how_measured_or_observed** (tooltip or expandable).
- Group by type (Aptitudes, Traits, Values) with small headings. Sorts: e.g. by mean descending, or by band (high first).

This gives clear evidence for the narrative and makes the summary feel grounded.

### 3. Show personality cluster

- Use **payload.personality_cluster.pre_survey_profile**.
- Display a short line or badge: e.g. “Based on your pre-survey: dominant **creative**, tone **adventurous**” (or whatever keys you use). No need to show the full object unless you add an “About this profile” expandable.

This ties the summary to “you and your pretest + answers”.

### 4. Optional “Your answers at a glance”

- Use **payload.questions_and_answers**.
- Compact list: question **title** + short representation of **userAnswer** (e.g. “Triangle: 50% A, 30% B, 20% C” or “Chose: Option 2”). Can be collapsible (“Show what you answered”) so the page is not long.

This reinforces that the summary is based on concrete choices.

### 5. “Skills that fit you” teaser

- Use **payload.skills** (already sorted by applicability).
- Show top 3–5: **name** + **applicability** (e.g. as a small bar or label). Link to “See all on Skills page” → `/skills` with sessionId in state.

This connects Profile Summary to Skills and careers without duplicating the full Skills page.

### 6. Fallback when LLM is missing or fails

- If `report.profileSummary` is null:
  - Fetch **payload** (if not already fetched).
  - Show a **structured fallback**:
    - “Your top dimensions” from payload.dimensions (e.g. top 5 by mean).
    - “Skills that fit you” (top 3–5 by applicability).
    - Personality cluster (pre_survey_profile).
  - Optional short line: “Narrative summary is being prepared; here’s what we measured.”

So the page never looks empty and always shows something useful from the payload.

### 7. Backend: optional single LLM call with payload

- If you want one LLM call for both (a) summary and (b) profession recommendations:
  - Backend endpoint (e.g. `POST /sessions/:sessionId/report/summarize`) that:
    - Builds the same payload (or calls `getSessionPayloadForLlm(sessionId)`).
    - Sends it to the LLM with a prompt that asks for summary + recommendations.
    - Returns { summary, recommendations }.
  - Profile Summary page then uses this response for the narrative and optionally links to a Recommendations view that uses the same recommendations.

This keeps the “single JSON representing the user” as the only input to the LLM, as intended.

---

## Suggested order of implementation

1. **Add `getReportPayload(sessionId)`** in `surveyApi.js` and document it.
2. **Profile Summary: fetch payload alongside report** (or instead of report when you only need structured data).
3. **Add “What we measured”** section from `payload.dimensions` (name, mean, band; optional description).
4. **Add “Skills that fit you”** teaser from `payload.skills` (top 3–5, link to Skills).
5. **Add personality cluster** line from `payload.personality_cluster.pre_survey_profile`.
6. **Fallback:** when both LLM summaries are null, show the structured fallback from payload (dimensions + skills + personality).
7. (Optional) **“Your answers at a glance”** collapsible from `payload.questions_and_answers`.
8. (Optional) **Backend:** single LLM endpoint that takes the payload and returns summary + recommendations.

This way the Profile Summary page is grounded in the same session JSON used for LLM summary and profession recommendations, and stays usable even when the LLM is slow or unavailable.
