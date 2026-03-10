# Report prompt design

Design for the LLM prompt used to generate the strength and personality report (profile summary and recommendations). Covers what the model receives, how it should use it, and how the payload is built.

## Purpose

- **Input:** User message starting with an optional one-line headline (profile, high dimensions, top skills) then the full JSON payload (session assessment summary); system prompt plus optional AI-era context.
- **Output:** Three HTML sections: "Your personality at a glance", "Study paths that could suit you", "Careers and opportunities that could benefit you most".
- **Readers:** Teenagers and young adults; tone is supportive, specific, and grounded in the data.

## Payload sections and necessity


| Section                                                   | Needed                     | Notes                                                                                                   |
| --------------------------------------------------------- | -------------------------- | ------------------------------------------------------------------------------------------------------- |
| **System prompt**                                         | Yes                        | Role, tone, output format, rules. Must state how to use band and pre_survey_profile.                    |
| **AI-era context**                                        | Yes (for resilient advice) | Aligns study/careers with what shrinks vs persists over 10-15 years.                                    |
| **questions_and_answers**                                 | Yes (once)                 | Scenario titles, prompts, vertices, choiceSummary (dimension names + shares); for triangle with dominant choice, chosenOptionLabel. No pre-written interpretation; model interprets from data. Single source; no duplicate. |
| **dimensions**                                            | Yes                        | id, name, description, band, mean, score_scale, and band_interpretation (precomputed for current band). |
| **skills**                                                | Yes                        | id, name, description, applicability; optional ai_trend for AI-era emphasis.                            |
| **personality_cluster.pre_survey_profile**                | Yes                        | Tone, complexity, demographics; model must respect toneInstruction and complexityInstruction.           |
| **personality_cluster.questions_and_answers_for_profile** | No                         | Removed; profile is based on the same assessment (questions_and_answers).                               |


## How the model should use the data

- **Dimensions:** Use `band` (low/medium/high) and `band_interpretation` (or `score_scale.interpretation[band]`) to phrase what this level means for that dimension. Reference dimension names in the report.
- **Study paths and careers:** Tie suggestions to both dimension strengths and high-applicability skills. Prefer 3-5 clear fits rather than long lists.
- **Personality at a glance:** Use at least two concrete scenarios from questions_and_answers (title, prompt, choiceSummary, chosenOptionLabel when present) so the reader recognizes themselves; do not only list dimension names.
- **Triangle placements (Option B):** The system prompt includes a short "How to interpret triangle placements" block drawn from bft-doc/triangle_assessment_methodology.md (rejection vs weak preference, zones: centre, edge, near-corner, call out excluded vertices). The model interprets from choiceSummary using these rules and can also reason over the whole assessment for cross-question patterns.
- **Pre-survey profile:** Respect toneInstruction, complexityInstruction, and demographics (e.g. high school) for wording and examples.
- **AI-era context:** Mention only when it clearly improves a suggestion (e.g. why a direction is resilient or what to avoid). Do not list generic AI-era points; weave in one or two per section where they fit the profile.

## Payload optimizations

- **No duplicate Q&A:** personality_cluster contains only pre_survey_profile.
- **Trimmed questions_and_answers:** Each item has title, type, prompt, vertices (label + dimensionId). For triangle questions: choiceSummary (dimension names and shares; no raw userAnswer, no pre-written interpretation so the model interprets from the data); for clear dominant, chosenOptionLabel. For non-triangle questions: userAnswer. questionId, description, and options omitted to reduce tokens.
- **Headline above JSON:** The user message may start with a one-line headline (profile, high dimensions, top skills) so the model has a quick anchor before the full JSON.
- **band_interpretation:** Each dimension includes a precomputed string for the current band so the model does not have to index into score_scale.interpretation.
- **Skills:** Sorted by applicability; optional ai_trend included when present for AI-era emphasis.

## Context size

The report call uses one system message (profile prompt + optional AI-era context) and one user message (headline + JSON). Total context is set by `LLM_NUM_CTX` (e.g. 32768). Typical usage: system prompt ~600 tokens, AI-era context ~800 tokens, headline ~30 tokens, payload ~5k-15k tokens (20-30 questions, dimensions, skills), output ~1-2k tokens. So 32k is enough for normal sessions (e.g. up to ~30 questions) with margin. If you raise the question cap or add much longer dimension text, monitor total input size or increase `LLM_NUM_CTX` / use a model with a larger window.

## Files

- **System prompt:** `bft-api/conf/report_profile_system_prompt.txt` (includes triangle interpretation rules from methodology).
- **Triangle methodology:** `bft-doc/triangle_assessment_methodology.md` (source for "How to interpret triangle placements" in the prompt).
- **AI-era context:** `bft-api/conf/report_ai_context.txt` (path set by LLM_REPORT_AI_CONTEXT_FILE)
- **Payload builder:** `bft-api/src/services/reportService.js` (`getSessionPayloadForLlm`)
- **Payload contract:** `bft-doc/session-payload-for-llm.md`

