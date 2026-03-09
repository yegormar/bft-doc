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
      "questionId": "scenario_0",
      "title": "...",
      "description": null,
      "type": "triangle",
      "prompt": "...",
      "vertices": { "a": { "label": "...", "dimensionId": "..." }, "b": {...}, "c": {...} },
      "userAnswer": { "a": 0.5, "b": 0.3, "c": 0.2 }
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
        "count": 2
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
      "applicability": 2.4
    }
  ],
  "personality_cluster": {
    "pre_survey_profile": { "dominant": [...], "secondary": "...", ... },
    "questions_and_answers_for_profile": [ /* same as questions_and_answers */ ]
  }
}
```

## Fields

- **session_id:** Session ULID.
- **questions_and_answers:** All questions served in the assessment and the user's answer per question. For triangle questions, `userAnswer` is barycentric `{ a, b, c }`; for single/multi choice, the selected value(s).
- **dimensions:** Measured dimensions grouped by type (aptitudes, traits, values). Each entry includes:
  - **id, name, description, how_measured_or_observed, related_skill_clusters, score_scale** from the assessment model.
  - **mean, band, count** from the calculated scores (mean 1–5, band low/medium/high, number of measurements).
- **skills:** All skills with an **applicability** score derived from dimension scores (traits, values, aptitudes). Sorted by applicability descending. Only **id, name, description, applicability** are included for the LLM.
- **personality_cluster:** Pre-survey (pretest) profile used for the session plus the same **questions_and_answers_for_profile** list, so the LLM can tie the assessment to the profile.

## Usage

1. Call `GET /api/sessions/:sessionId/report/payload` after the assessment is complete (or at any time; partial data is returned if in progress).
2. Send the response body as input to your LLM prompt for (a) assessment summary and (b) profession recommendations.
