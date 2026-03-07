# Scenario batches, design instructions, and rank questions

This document describes the batch-based interview flow (reserved for future use), the scenario design instructions used for LLM-generated questions, and the **rank** question type.

---

## Scenario batches (future or alternate flow)

- **File:** `bft-api/src/data/scenarioBatches.json`
- **Status:** The file is kept for future or alternate flow. The current app does **not** use batch selection; it uses per-dimension selection (`selectOneDimensionRandom`) and dimension-based completion. Batch selection and related state (`usedBatchIds`, batch-themed pregen) were removed to avoid dead code.
- **Purpose (when wired):** Predefined batches of traits and values (2–3 dimensions per batch). The API would select one batch at a time to request a single scenario question that probes those dimensions together, with a fixed question cap (e.g. from `constraints.maxQuestionsPerInterview`).

### How selection would work (when implemented)

1. **When batches exist:** `assessmentService` would use a batch selector (e.g. `selectNextBatch(state, model)`) instead of per-dimension coverage selection.
2. **Selection order:** Batches are chosen by (1) least times already used, (2) then by least total coverage of that batch’s dimensions. So under-covered batches are preferred.
3. **State:** Each session tracks `usedBatchIds` (list of batch ids already served). Completion is based on **question count**, not per-dimension coverage.
4. **Max questions:** When using batches, the effective maximum is `MAX_INTERVIEW_QUESTIONS` (env) or, if unset, `constraints.maxQuestionsPerInterview` from the batches file (20). The interview is complete when `questionIndex >= effectiveMax`.
5. **Progress:** Progress is reported as questions asked vs that max (e.g. “5 / 20 questions”, percentComplete = 25).

### Batch file shape

- `batches`: array of `{ id, theme?, dimensions: [{ dimensionType, dimensionId }], preferredResponseType? }`
- `constraints`: `maxQuestionsPerInterview` (number), `allowedResponseTypes`, etc.
- Dimension ids must match `traits.json` and `values.json`. The service enriches each batch’s dimensions with `name`, `question_hints`, and `how_measured_or_observed` from the assessment model before sending to the question generator.

### Fallback

If `scenarioBatches.json` is missing or `batches` is empty, the service falls back to the previous behavior: `selectNextDimensionSet(coverage, model)` and dimension-based completion (all dimensions meet `minSignalPerDimension`).

---

## Scenario generation: three-step pipeline

- **File (Step 1):** `bft-api/conf/scenario_step1.txt`
- **Purpose (Step 1):** Generate **one** scenario in **plain text only** (no JSON). Output uses headings TITLE:, SITUATION:, OPTIONS:. One of three strategies; no dimension names or obvious synonyms. **Step 2:** Critique then judge (two LLM calls in code; if judge says YES, retry Step 1 up to 3 times). **Step 3:** `scenario_step2.txt` converts the one scenario to JSON and assigns dimensionScores.

---



---

## Rank question type

- **Meaning:** The user **orders** the options from “most like me” to “least like me”. The submitted value is an **ordered array** of option `value` strings (e.g. `["a", "c", "b"]`).

### API

- **Validation:** `ollamaInterview.validateNextQuestionObject()` accepts `type: "rank"`. For rank, `maxSelections` is omitted; options are the list to be ordered.
- **Storage:** Answers are stored as `{ questionId, value: string[] }`. Reports and `getAssessment` return that array as-is.
- **Batches:** A batch can set `preferredResponseType: "rank"`; the generator uses it as a hint when calling the LLM.

### UI

- **MainSurvey:** Initial value for a rank question is `question.options.map(o => o.value)` (default order). Submitting sends the current ordered array. `canProceed` requires the array length to equal `question.options.length`.
- **PreSurveyQuestion:** For `question.type === 'rank'`, options are shown in the current order with “Move up” / “Move down” buttons to reorder. Value is always the ordered list of option values.

### Analytics / results

- **Results page:** If an insight’s `value` is an array, it is displayed as “You ranked (most to least):” with a numbered list. Otherwise “You chose: …” is shown.
- **clusterProfile:** Used only for the pre-survey; main-survey rank answers are not used there.

---

## Related files

| File | Role |
|------|------|
| `bft-api/src/data/scenarioBatches.json` | Batch definitions (traits/values per batch, preferredResponseType, constraints). |
| `bft-api/conf/scenario_step1.txt` | Step 1 creative prompt (one plain-text scenario). |
| `bft-api/conf/scenario_step2.txt` | Step 3 format and score prompt (JSON + dimensionScores). |
| `bft-api/conf/README.md` | Short pointer to both; wiring of scenario prompt. |
| `bft-api/src/services/assessmentService.js` | `getScenarioBatches`, `selectNextBatch`, `getEffectiveMaxQuestions`, batch-based completion and progress. |
| `bft-api/src/lib/ollamaInterview.js` | Three-step flow: `generateOneScenarioPassingCritique`, `runCritiquePass`, `formatAndScoreScenario`; rank in `VALID_TYPES`; `parseStep1PlainText`. |
| `bft-ui/.../PreSurveyQuestion.jsx` | Rank UI (ordered list, up/down). |
| `bft-ui/.../MainSurvey.jsx` | Rank initial value, submit payload, canProceed. |
| `bft-ui/.../ResultsPage.jsx` | Display of rank answers (array as numbered list). |

---

## Testing

- **API:** See `bft-doc/QUESTION_GENERATION_TEST_COVERAGE.md`. New tests cover: batch loading and selection when batches exist, completion at max questions (batch mode), and rank validation in `ollamaInterview`.
- **UI:** Rank flow is manual-tested (reorder options, submit, check results page). No UI test runner is configured for rank specifically.
