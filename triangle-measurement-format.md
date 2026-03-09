# Triangle Measurement Format

This document describes how a triangle assessment response is captured, transmitted, and converted into dimension scores in the BFT system. It is the technical reference for the triangle measurement pipeline.

---

## 1. Overview

1. The user drags a ball on a triangle. The ball position is represented as **barycentric coordinates** `(a, b, c)` where `a + b + c = 1` and each value is in `[0, 1]`.
2. Vertex **A** is the top of the triangle, **B** bottom-left, **C** bottom-right. Each vertex is bound to a **dimension** (trait or value) in `dimension_triangles.json`.
3. The coordinate for a vertex is the weight of that corner in the position: `a` for vertex A, `b` for B, `c` for C. A ball at vertex A has `(a, b, c) = (1, 0, 0)`.
4. Each coordinate is converted to a **1-to-5 score** with the formula below. One triangle thus yields three dimension scores (one per vertex).
5. When a dimension appears in more than one triangle, scores are **aggregated by simple average** (sum of scores / count of measurements).

---

## 2. Vertex Order and Dimension Mapping

- In the UI and in the API, vertices are always **a**, **b**, **c** in that order.
- In `dimension_triangles.json`, each triangle has `vertices.a`, `vertices.b`, `vertices.c`. Each vertex has:
  - `label`: short text shown at the corner (e.g. "The tangible, positive change in the lives of the people you served").
  - `dimensionId`: the dimension being measured at that corner (e.g. `value_helping_others_impact`).
- The API maps vertex order to dimensions when building the question: `dims[0]` = dimension at A, `dims[1]` = B, `dims[2]` = C. The same order is used when applying the answer: score for `dims[0].id` comes from coordinate `a`, and so on.

**Validation:** The backend builds `dimensionSet` from `triangle.vertices` in order `['a','b','c']`, so the first dimension in the set is always the one at vertex A, the second at B, the third at C.

---

## 3. Capture in the UI (bft-ui)

- The triangle is drawn in **normalized coordinates** (0 to 1) with fixed vertex positions:
  - A: (0.5, 0.12)
  - B: (0.08, 0.88)
  - C: (0.92, 0.88)
- On pointer/touch move, the pointer position `(px, py)` (in the same 0-1 space relative to the SVG) is converted to barycentric `(a, b, c)` using the **area method**: each coordinate is proportional to the area of the sub-triangle opposite that vertex (e.g. `a` = area(PBC) / area(ABC)).
- The result is **clamped** so each of `a`, `b`, `c` is in `[0, 1]` and renormalized so `a + b + c = 1` (nearest point inside the triangle).
- The UI sends to the API a value of the form:
  ```json
  { "a": 0.65, "b": 0.20, "c": 0.15 }
  ```
  Values are rounded to three decimal places. Sum must be 1 (within floating point).

**Payload:** When the user clicks "Next", the frontend sends:

```http
POST /api/sessions/:sessionId/assessment/answers
Content-Type: application/json

{ "answers": [ { "questionId": "scenario_0", "value": { "a": 0.65, "b": 0.2, "c": 0.15 } } ] }
```

---

## 4. Receipt and Normalization in the API (bft-api)

- The assessment controller calls `assessmentService.submitAnswers(sessionId, body)`. The answer is stored and then `applyOneAnswerToState(state, answer, sessionId)` is run.
- For `served.type === 'triangle'`:
  1. **Read value:** `rawValue = answer.value` (expected to be `{ a, b, c }`). If missing or invalid, fallback is `(1/3, 1/3, 1/3)`.
  2. **Clamp and normalize:** Each of `a`, `b`, `c` is forced into `[0, 1]`, then divided by their sum so that `na + nb + nc = 1`. This ensures valid barycentric even if the client sent slightly off values.
  3. **Look up dimension set:** `dims = state.questionToDimension[qid]` is the array of three dimensions for this triangle, in order A, B, C (from when the question was served).

---

## 5. Score Formula

For each vertex, the barycentric coordinate is converted to a **1-to-5 scale**:

```
score = (coordinate × 4) + 1
```

Constants in code: `TRIANGLE_SCORE_SCALE = { mult: 4, add: 1 }`.


| Barycentric coordinate | Score | Interpretation                      |
| ---------------------- | ----- | ----------------------------------- |
| 0.0                    | 1.0   | No affinity (ball at opposite edge) |
| 0.25                   | 2.0   | Low affinity                        |
| 0.33                   | 2.33  | Neutral (centre of triangle)        |
| 0.5                    | 3.0   | Moderate affinity                   |
| 0.75                   | 4.0   | High affinity                       |
| 1.0                    | 5.0   | Full affinity (ball at vertex)      |


So for one triangle answer, three scores are produced:

- `scoreA = na * 4 + 1` for `dims[0].id`
- `scoreB = nb * 4 + 1` for `dims[1].id`
- `scoreC = nc * 4 + 1` for `dims[2].id`

Scores are rounded to two decimal places before being added to the aggregate.

---

## 6. Gain vs Loss Framing

- Each triangle has a `framing` field in the JSON: `"gain"` or `"loss"`.
- **Gain:** Proximity to a vertex means higher affinity with that dimension. The formula above is used as-is.
- **Loss:** Proximity to a vertex means "the absence of this is most painful", which still maps to a **high score** on that dimension. The same formula is used: no sign flip. The prompt and vertex labels convey the loss framing; the scoring is identical so that a person who is most pained by the absence of X gets a high score on X.

No extra logic in the code differentiates gain and loss for the math; both use `score = (coord * 4) + 1`.

---

## 7. Aggregation Across Triangles

- The backend keeps a **dimension scores aggregate** per session: `state.dimensionScoresAggregate = { traits: {}, values: {} }`.
- For each dimension id, the aggregate stores `{ sum, count }`.
- When a triangle answer is applied, `addDimensionScoresToAggregate(aggregate, dims, scoresByDimensionId)` is called. For each of the three dimensions, it does:
  - `bucket[id].sum += score`
  - `bucket[id].count += 1`
- So if a dimension is measured in two triangles, it will have `count === 2` and `sum` equal to the sum of the two scores. The **reported score** for that dimension is the **mean**: `mean = sum / count`, rounded to two decimals.
- Only **traits** and **values** are aggregated; aptitudes and skills are not updated from triangle answers.

---

## 8. Final Dimension Score Output (triangle mode: Bayesian scorer)

- In **triangle mode**, dimension scores are produced by the **Bayesian latent-trait scorer** (`bft-api/src/lib/bayesianTriangleScorer.js`), not by simple average of per-triangle scores.
- The scorer fits one latent strength (theta) per dimension using all triangle responses jointly:
  - **Dirichlet likelihood** with zone-aware concentration (corner, near_corner, edge, near_edge, centre).
  - **Rejection term**: low-weight vertices (< 0.15) add explicit evidence that that dimension is weaker than the dominant one.
- Inference: Metropolis-Hastings MCMC. Scores are mapped from posterior mean theta to [1, 5] preserving rank order.
- Output shape is unchanged: `{ traits: [], values: [], aptitudes: [] }` with `id`, `name`, `mean`, `band`, `count`. Band: `mean <= 2` → `"low"`, `mean >= 4` → `"high"`, else `"medium"`.
- The in-memory aggregate (sum/count) is still updated when answers are submitted (for compatibility); the **returned** dimension scores in getAssessment and getSessionHealth come from the Bayesian scorer when there is at least one triangle answer.

---

## 9. Validation Checklist

- **Vertex order:** UI and API both use a, b, c in the same order; backend builds `dimensionSet` from `vertices.a`, `vertices.b`, `vertices.c` so that index 0 = A, 1 = B, 2 = C.
- **Barycentric:** UI clamps and normalizes before send; API clamps and renormalizes again. Coordinates are in [0, 1] and sum to 1.
- **Formula:** Single formula everywhere: `score = (coord * 4) + 1`, no special case for loss framing.
- **Aggregation:** Simple average (sum/count) across all triangle (and scenario) measurements per dimension.
- **Ids:** `dimensionId` in the JSON (e.g. `value_helping_others_impact`) is the same as the key in the aggregate and in the final output; dimension type is inferred from the id prefix (`value_` or `trait_`).

---

## 10. Code References


| Step                                  | Location                                                                                                              |
| ------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| Triangle data and vertex-to-dimension | `bft-api/src/data/dimension_triangles.json`                                                                           |
| Serve triangle, build dimensionSet    | `bft-api/src/services/assessmentService.js`: `getNextTriangleQuestion`, `triangleToDimensionSet`                      |
| Apply answer, normalize, score        | `bft-api/src/services/assessmentService.js`: `applyOneAnswerToState` (triangle branch), `TRIANGLE_SCORE_SCALE`        |
| Add to aggregate                      | `bft-api/src/services/assessmentService.js`: `addDimensionScoresToAggregate`                                          |
| Triangle mode: dimension scores       | `bft-api/src/lib/bayesianTriangleScorer.js`: `scoreTriangleResponses` (zone-aware Dirichlet + rejection, MCMC)       |
| Build output (mean, band)             | `bft-api/src/services/assessmentService.js`: `buildDimensionScoresForAssessment`, `buildDimensionScoresOutputFromBayesian` |
| UI: pointer to barycentric            | `bft-ui/src/components/Discovery/TriangleQuestion.jsx`: `cartesianToBarycentric`, `clampBarycentric`, `handlePointer` |
| UI: submit value                      | `bft-ui/src/components/Discovery/MainSurvey.jsx`: payload `value` as `{ a, b, c }`                                    |


Methodology and design rationale (why triangles, gain/loss, etc.) are in **bft-doc/triangle_assessment_methodology.md**.

Tests that assert this behaviour (with `BFT_ASSESSMENT_MODE=triangles`) are in **bft-api/test/assessmentService.triangle.test.js**.