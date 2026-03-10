# Session payload for LLM

This document describes the JSON object returned by `GET /api/sessions/:sessionId/report/payload`. It is intended to be submitted to an LLM for:

- (a) Summary of the assessment
- (b) Recommendations about profession selection

## Endpoint

- **Method and path:** `GET /api/sessions/:sessionId/report/payload`
- **Response:** Single JSON object (application/json). 404 if session not found.

## Payload shape

```json
{
  "session_id": "<sessionId>",
  "questions_and_answers": [
    {
      "title": "...",
      "type": "triangle",
      "prompt": "...",
      "vertices": { "a": { "label": "...", "dimensionId": "..." }, "b": {...}, "c": {...} },
      "choiceSummary": "Position: Pattern recognition and abstraction 50%, Quantitative and numerical facility 30%, Technical and hands-on 20%",
      "chosenOptionLabel": "The option label they leaned to (only when there is a clear dominant vertex)."
    }
  ],
  "dimensions": {
    "aptitudes": [
      {
        "id": "aptitude_...",
        "name": "...",
        "description": "...",
        "how_measured_or_observed": "...",
        "related_skill_clusters": ["skill_id_1", "..."],
        "score_scale": { "min": 1, "max": 5, "interpretation": {...} },
        "mean": 3.5,
        "band": "medium",
        "count": 2,
        "band_interpretation": "Precomputed string for this band (from score_scale.interpretation[band])."
      }
    ],
    "traits": [ /* same shape */ ],
    "values": [ /* same shape */ ]
  },
  "skills": [
    {
      "id": "skill_id",
      "name": "...",
      "description": "...",
      "applicability": 2.4,
      "ai_trend": "grows"
    }
  ],
  "personality_cluster": {
    "pre_survey_profile": { "dominant": [...], "secondary": "...", "toneInstruction": "...", "complexityInstruction": "...", "demographics": {...}, ... }
  }
}
```

The profile is based on the same assessment (questions_and_answers). personality_cluster no longer duplicates questions_and_answers.

## Fields

- **session_id:** Session ULID.
- **questions_and_answers:** All questions served in the assessment and the user's answer per question. Trimmed for the LLM: title, type, prompt, vertices; for triangle questions choiceSummary only (no userAnswer, no pre-written interpretation). For triangle questions with a clear dominant vertex, **chosenOptionLabel** is included. For non-triangle questions, userAnswer is included. questionId, description, and options are omitted. The model interprets strength and balance from the shares itself. For triangle questions:
  - **choiceSummary:** Dimension names and shares (e.g. "Position: Pattern recognition 33%, Quantitative 33%, Technical hands-on 33%"). The model sees which dimension got which share and interprets strength/balance from this data; no pre-written interpretation is sent.
  - **chosenOptionLabel:** When the answer has a clear dominant vertex (not balanced), the label of that option so the model can reference the concrete choice for narrative.
- **dimensions:** Measured dimensions grouped by type (aptitudes, traits, values). Each entry includes:
  - **id, name, description, how_measured_or_observed, related_skill_clusters, score_scale** from the assessment model.
  - **mean, band, count** from the calculated scores (mean 1-5, band low/medium/high, number of measurements).
  - **band_interpretation:** Precomputed string from score_scale.interpretation[band] so the LLM can phrase what this band means for this dimension without indexing.
- **skills:** All skills with an **applicability** score derived from dimension scores (traits, values, aptitudes). Sorted by applicability descending. **id, name, description, applicability**; **ai_trend** (grows/stays/mixed) when present in the skill definition.
- **personality_cluster:** Only **pre_survey_profile** (toneInstruction, complexityInstruction, demographics, dominant, secondary, etc.). The profile is based on the same assessment; questions_and_answers is not duplicated here. See bft-doc/report-prompt-design.md.

## Usage

1. Call `GET /api/sessions/:sessionId/report/payload` after the assessment is complete (or at any time; partial data is returned if in progress).
2. For the report LLM call, the app prepends a one-line headline (profile, high dimensions, top skills) above the JSON, then sends the combined user message. Send the response body (or headline + JSON) as input to your LLM for (a) assessment summary and (b) profession recommendations.
