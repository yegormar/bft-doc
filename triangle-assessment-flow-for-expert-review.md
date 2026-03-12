# Triangle Assessment: End-to-End Flow for Expert Review

## Objective

To give an external expert everything needed to assess the triangle assessment: the three-step pipeline (capture, per-triangle interpretation, aggregation), how each step is implemented in code, and how design intent (e.g. aptitudes measured among themselves, traits and values combined) compares with the current implementation. The expert can use this document to evaluate methodology, scoring, and any gaps.

## Context

- **BFT** (Build for Tomorrow) is a career and study guidance product. Users complete an assessment (triangle mode: drag a ball on triangles; each corner is a dimension such as a value or trait). The system produces dimension scores (1-5, bands) and a report.
- **Triangle assessment** uses barycentric coordinates (a, b, c summing to 1), zone-based interpretation (corner, edge, centre, etc.), and a Bayesian latent-trait scorer to aggregate many triangle answers into one score per dimension. Triangles are defined in two data files: one for traits and values, one for aptitudes; they are merged into a single question sequence.
- **Audience:** A reviewer who may not have the codebase open; the doc therefore cites file paths and line ranges and includes concrete examples from the app (tutorial triangle, real triangle, scoring formula).

**Related docs:** `bft-doc/triangle_assessment_methodology.md` (design rationale, zones, rejection), `bft-doc/triangle-measurement-format.md` (technical format, API payloads), `bft-doc/triangle-aptitudes-vs-traits-values.md` (aptitudes vs traits/values: design intent vs implementation).

---

## Overview: Three Steps

| Step | What it does | Output |
|------|--------------|--------|
| **1. Capture and build triangles** | User drags ball; UI sends barycentric (a, b, c); API normalizes and maps to dimensions. | Stored answer per question; per-triangle three coordinates and three dimension IDs. |
| **2. Per-triangle interpretation** | Zone is detected from (a, b, c); rejection is applied when a weight &lt; 0.15; a human-readable summary is produced. | "Position: A 27%, B 64%, C 9%" and text like "B is your main pull; A is in play. C is ruled out." |
| **3. Aggregation and final scores** | Raw responses are split into aptitude-only and trait+value; the Bayesian scorer runs separately on each group so the 1-5 scale is relative within each group. A simple zone-weighted average is also computed and compared (logged) for a future simplify decision. | One score per dimension (mean, band) for traits, values, and aptitudes. |

Steps 1 and 2 do **not** require the Bayesian model. Step 3 is where many triangles are combined into one strength per dimension.

---

## Step 1: Capture and Build Triangles

### What happens

1. The user sees a triangle with three vertices. Each vertex has a **label** (e.g. "You did something most people couldn't") and a **dimensionId** (e.g. `value_mastery_growth`). Data comes from `dimension_triangles.json`.
2. The user drags a ball. The UI converts pointer position to **barycentric coordinates** (a, b, c) with a + b + c = 1, each in [0, 1]. The same order is used everywhere: vertex A = first dimension, B = second, C = third.
3. On submit, the frontend sends one answer per triangle, e.g. `{ "questionId": "scenario_0", "value": { "a": 0.65, "b": 0.2, "c": 0.15 } }`.
4. The API receives the answer, clamps and renormalizes (a, b, c), and looks up the three dimensions for that question from session state. No scoring or interpretation yet; the aggregate is updated with the **raw formula** score for compatibility, but in triangle mode the **returned** dimension scores come from Step 3.

### Example: Triangle "The Meaning of a Win" (triangle_01)

From `bft-api/src/data/dimension_triangles.json`:

- **Prompt:** "You just nailed something - an exam, a competition, a personal goal. Which feeling hits hardest?"
- **Vertices:**  
  - A: "You actually helped someone" → `value_helping_others_impact`  
  - B: "You did something most people couldn't" → `value_mastery_growth`  
  - C: "People saw you do it" → `value_recognition_visibility`

**User placement:** Ball near the Mastery corner: (a, b, c) = (0.15, 0.75, 0.10).

After normalization (already valid), the stored answer is `{ a: 0.15, b: 0.75, c: 0.10 }`. The backend knows this question maps to dimensions `[value_helping_others_impact, value_mastery_growth, value_recognition_visibility]` in order A, B, C.

### Code references (Step 1)

| Role | Location |
|------|----------|
| Triangle definitions, vertex labels, dimensionIds | `bft-api/src/data/dimension_triangles.json` |
| Build dimension set for a triangle (A, B, C order) | `bft-api/src/services/assessmentService.js`: `triangleToDimensionSet` (lines 83-94) |
| Serve next triangle, assign questionId, store questionToDimension | `assessmentService.js`: `getNextTriangleQuestion` (lines 114-174) |
| Apply answer: read value, clamp, normalize, score formula, add to aggregate | `assessmentService.js`: `applyOneAnswerToState` (lines 469-488); `TRIANGLE_SCORE_SCALE = { mult: 4, add: 1 }` (line 48) |
| UI: pointer to barycentric, clamp, submit value | `bft-ui/src/components/Discovery/TriangleQuestion.jsx`: cartesianToBarycentric, clampBarycentric, handlePointer; `MainSurvey.jsx`: payload `value` as `{ a, b, c }` |

**Score formula (used in aggregate and in per-triangle display):** `score = (coordinate × 4) + 1`, so coordinate 0 → 1, 0.33 → 2.33, 1 → 5.

---

## Step 2: Per-Triangle Interpretation (Zones and Rejection)

### What happens

For each triangle answer, the system:

1. **Detects the zone** from the normalized (a, b, c) using fixed thresholds (see table below). Zones: corner, near_corner, edge, near_edge, centre.
2. **Rejection:** If any vertex has weight &lt; 0.15, that dimension is treated as **explicitly rejected** ("not in play"), not as a "low score."
3. **Human-readable summary:** Text is generated in the style of the methodology: main pull, in play, ruled out; or blended; or weak evidence (centre).

This step does **not** use the Bayesian model. It only needs the single placement and the zone/rejection rules.

### Zone detection (same logic in backend scorer and UI)

Implementation: sort weights descending (max, mid, min). Then:

| Zone | Condition (from code) |
|------|-----------------------|
| **corner** | max ≥ 0.7 |
| **near_corner** | min &lt; 0.2 and max ≥ 0.55 |
| **edge** | min &lt; 0.2 and mid in [0.35, 0.65] and max and mid close (|max - mid| &lt; 0.2) |
| **near_edge** | min &lt; 0.2 and not edge |
| **centre** | otherwise (no vertex strongly rejected) |

Code: `bft-api/src/lib/bayesianTriangleScorer.js`: `detectZone` (lines 42-60). Same thresholds in `bft-ui`: `TriangleExplainerPage.jsx` and `ResultsAnswersPage.jsx`.

### Example: Tutorial triangle (Explainer)

The **explainer** uses a hypothetical question (no data sent to server):

- **Prompt:** "You're having people over. What do you offer?"
- **Vertices:** A = "Desserts & treats", B = "Full meal", C = "Snacks & bites"

**Placement (0.65, 0.28, 0.07):** max = 0.65, min = 0.07. So min &lt; 0.2 and max ≥ 0.55 → **near_corner**. The third vertex is under 0.15 → rejected.

**Interpretation (from `TriangleExplainerPage.jsx`):**

- Zone label: "Near corner"
- Points: "A is your main choice, but you kept B in the picture." / "You did not go all the way to only A." / "C is out; you are not offering that."
- Closing: "How the system reads it: strong pull toward A, B still counts, C is treated as rejected."

### Example: Real triangle (Results page)

Same placement (0.65, 0.28, 0.07) on **triangle_01** (Helping others = A, Mastery = B, Recognition = C). So B is dominant, A is secondary, C is rejected.

**Display:** "Position: B 65%, A 28%, C 7%"

**Summary (from `ResultsAnswersPage.jsx`):**  
"Mastery growth is your main pull; Helping others impact is in play. Recognition visibility is ruled out (you left it out of your choice, not a small preference for it)."

### Code references (Step 2)

| Role | Location |
|------|----------|
| Zone detection (backend) | `bft-api/src/lib/bayesianTriangleScorer.js`: `detectZone` |
| Zone detection and summary for results | `bft-ui/src/components/pages/ResultsAnswersPage.jsx`: `detectZone`, `triangleInterpretation` (lines 34-106) |
| Tutorial explainer: zone and "You are saying" text | `bft-ui/src/components/Discovery/TriangleExplainerPage.jsx`: `detectZone`, `getPlacementExplanation` (lines 30-157) |
| Report: LLM instructed to use zone-based language and rejection | `bft-api/conf/report_profile_system_prompt.txt`: "How to interpret triangle placements" (low share = rejection, centre = weak evidence, edge = blended, corner = dominant) |

---

## Step 3: Aggregation and Final Dimension Scores

### What happens

When the assessment is read (e.g. getAssessment), the system:

1. **Collects raw triangle responses** from session state and answers: for each triangle answer, one object `{ dims: [dimIdA, dimIdB, dimIdC], weights: [a, b, c] }`. Zone is computed inside the scorer from weights (or can be passed).
2. **Splits responses by type:** Aptitude-only triangles (all three dims start with `aptitude_`) go into one list; all other triangles (traits and values) go into another. See **bft-doc/triangle-aptitudes-vs-traits-values.md** for design intent.
3. **Runs the Bayesian latent-trait scorer twice:** Once on the aptitude list, once on the trait+value list. Each run has one latent strength (θ) per dimension in that list; likelihood is Dirichlet(weights | φ(zone) × softmax(θ)) plus a rejection term for vertices with weight &lt; 0.15. Prior: θ ~ Normal(0, 1.5) per dimension. Inference: Metropolis-Hastings MCMC (2000 warmup, 8000 samples). Each run maps posterior mean θ to 1-5 **within that group only** (`lo` and `hi` are taken over that run's dimensions). The two profiles are merged into one list.
4. **Builds output:** Each dimension gets `mean` (1-5), `band` (low if mean ≤ 2, high if mean ≥ 4, else medium), and `count` (number of triangles that included it). So one aptitude and one trait/value can both be 5.0 (each is strongest in its own group).
5. **Comparison logging (for future simplify decision):** The system also computes a **simple zone-weighted average** (same raw scores, weighted by zone, then rescale to 1-5 within each group). It logs MCMC vs simple: Spearman rank correlation, whether top dimension agrees, whether bottom dimension agrees, and the dimension ids. If they agree often in practice, the simpler path could replace MCMC later.

So Step 3 is the only place that combines multiple triangles into one strength per dimension. The 1-5 scale is **relative within each group** (aptitudes among themselves; traits and values among themselves).

### Example (conceptual)

After split: aptitude list has 6 dimensions, trait+value list has 12. MCMC run on aptitudes: one aptitude gets 5.0, one gets 1.0. MCMC run on traits+values: one trait gets 5.0, one value gets 1.0. Merged output: both the strongest aptitude and the strongest trait/value can have mean 5.0. "Level 5" means "strongest in this group," not "strongest overall."

### Code references (Step 3)

| Role | Location |
|------|----------|
| Collect answers into raw responses | `bft-api/src/services/assessmentService.js`: `collectTriangleRawResponses` |
| Split by type (aptitude vs trait+value) | `assessmentService.js`: `splitRawResponsesByType` |
| Bayesian scorer (called twice) | `bft-api/src/lib/bayesianTriangleScorer.js`: `scoreTriangleResponses`, `thetaToScore15` |
| Simple zone-weighted average (for comparison) | `assessmentService.js`: `simpleZoneWeightedScores`, `ZONE_WEIGHT` |
| MCMC vs simple comparison logging | `assessmentService.js`: `logMcmcVsSimpleComparison` |
| Build final dimension scores | `assessmentService.js`: `buildDimensionScoresForAssessment`, `buildDimensionScoresOutputFromBayesian` |

---

## Summary Table for Expert

| Step | Purpose | Needs Bayesian? | Main code |
|------|----------|-------------------|-----------|
| 1 | Store placement (a,b,c) and dimension IDs per question; optional aggregate update | No | `assessmentService.js`: triangleToDimensionSet, applyOneAnswerToState |
| 2 | Show "Position: A x%, B y%, C z%" and zone-based text (main pull, in play, ruled out) | No | `ResultsAnswersPage.jsx`: detectZone, triangleInterpretation; `TriangleExplainerPage.jsx`: getPlacementExplanation |
| 3 | One score per dimension (mean 1-5, band); separate scaling (aptitudes; traits+values); MCMC vs simple logged | Yes | `assessmentService.js`: splitRawResponsesByType, buildDimensionScoresForAssessment, simpleZoneWeightedScores, logMcmcVsSimpleComparison; `bayesianTriangleScorer.js`: scoreTriangleResponses |

---

## Appendix: Constants and Formula

- **Score from coordinate:** `score = (coordinate × 4) + 1`; scale [0, 1] → [1, 5].  
  Code: `assessmentService.js` line 48 `TRIANGLE_SCORE_SCALE`, lines 479-484.
- **Rejection threshold:** weight &lt; 0.15 → dimension "ruled out."  
  Code: `bayesianTriangleScorer.js` line 20 `REJECTION_THRESHOLD`; UI same logic in ResultsAnswersPage (third.w &lt; REJECTION_WEIGHT_THRESHOLD).
- **Zone concentration (φ):** corner 12, near_corner 9, edge 5, near_edge 4, centre 2.  
  Code: `bayesianTriangleScorer.js` lines 22-28 `ZONE_PHI`.
- **Rejection strength by zone:** corner 1.0, near_corner 0.9, edge/near_edge 0.7, centre 0.0.  
  Code: `bayesianTriangleScorer.js` lines 30-36 `REJECTION_STRENGTH`.
- **MCMC:** 2000 warmup iterations (discarded), 8000 iterations kept; step size adapted during warmup.  
  Code: `bayesianTriangleScorer.js` `runMcmc` (lines 157-182).
- **Zone weights (simple path):** corner 1, near_corner 0.9, edge 0.7, near_edge 0.7, centre 0.3.  
  Code: `assessmentService.js` `ZONE_WEIGHT`.

---

*Document version: 1.1. Separate scaling (aptitudes vs traits+values) and MCMC vs simple comparison logging implemented. For methodology rationale, see bft-doc/triangle_assessment_methodology.md.*
