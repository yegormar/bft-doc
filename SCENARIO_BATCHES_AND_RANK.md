# Scenario batches, design instructions, and rank questions

This document describes the batch-based interview flow, the scenario design instructions used for LLM-generated questions, and the **rank** question type.

---

## Scenario batches (offline preparation)

- **File:** `bft-api/src/data/scenarioBatches.json`
- **Purpose:** Predefined batches of traits and values (2–3 dimensions per batch). During the interview, the API selects one batch at a time to request a single scenario question that probes those dimensions together. This keeps scenarios coherent and caps the interview at a fixed number of questions (default 20).

### How selection works

1. **When batches exist:** `assessmentService` uses `selectNextBatch(state, model)` instead of per-dimension coverage selection.
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

## Scenario design instructions (LLM prompt)

- **File:** `bft-api/conf/scenario_design_instructions.txt`
- **Purpose:** Instructions so that scenario questions are **hard to reverse-engineer**: indirect probing, short descriptions (2–3 sentences), short options, and no dimension names or obvious synonyms in the question or options.

### How it’s used

- When generating a scenario question, `ollamaInterview.getScenarioSystemPrompt()` **prepends** the contents of this file to the base scenario system prompt. So the LLM receives both the design instructions and the response-format rules.
- If the file is missing or unreadable, only the base prompt is used.

### Main rules (from the file)

- Describe a **concrete situation** (dilemma, trade-off); options = **what the person would do**, not self-labels.
- **Do not** use dimension labels or obvious proxies in scenario or options.
- **Length:** Scenario description 2–3 sentences; each option one short sentence or phrase.
- **Response types:** Prefer `single_choice`, `multi_choice`, or `rank` as appropriate; the batch’s `preferredResponseType` is passed as a hint in the user prompt.

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
| `bft-api/conf/scenario_design_instructions.txt` | Design instructions prepended to scenario system prompt. |
| `bft-api/conf/README.md` | Short pointer to both; wiring of scenario prompt. |
| `bft-api/src/services/assessmentService.js` | `getScenarioBatches`, `selectNextBatch`, `getEffectiveMaxQuestions`, batch-based completion and progress. |
| `bft-api/src/lib/ollamaInterview.js` | `getScenarioSystemPrompt()`, rank in `VALID_TYPES`, `buildScenarioUserPrompt(..., preferredResponseType)`. |
| `bft-ui/.../PreSurveyQuestion.jsx` | Rank UI (ordered list, up/down). |
| `bft-ui/.../MainSurvey.jsx` | Rank initial value, submit payload, canProceed. |
| `bft-ui/.../ResultsPage.jsx` | Display of rank answers (array as numbered list). |

---

## Testing

- **API:** See `bft-doc/QUESTION_GENERATION_TEST_COVERAGE.md`. New tests cover: batch loading and selection when batches exist, completion at max questions (batch mode), and rank validation in `ollamaInterview`.
- **UI:** Rank flow is manual-tested (reorder options, submit, check results page). No UI test runner is configured for rank specifically.
