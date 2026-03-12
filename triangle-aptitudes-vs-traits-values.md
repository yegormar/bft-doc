# Aptitudes vs traits and values: design intent vs implementation

## Objective

To state the intended design (aptitudes measured among themselves; traits and values measured combined), to document the current implementation (separate scaling is implemented), and to note the MCMC vs simple comparison and possible future simplification. This supports expert review and product decisions.

## Context

- In the triangle assessment, **dimensions** are of three types: **aptitudes** (capacity, potential), **traits** (behavioural tendencies), and **values** (what one optimises for). Triangles are defined in two files: `dimension_triangles.json` (traits and values only; some triangles mix both) and `dimension_triangles_aptitudes.json` (aptitudes only). No triangle mixes an aptitude with a trait or value.
- The **aggregation step** (Step 3) runs a Bayesian latent-trait scorer and maps posterior mean θ to a 1-5 scale using a **relative** mapping within each group. Separate scaling (aptitudes among themselves; traits and values combined) is **implemented**. The system also computes a simple zone-weighted average and logs comparison (Spearman, top/bottom agreement) for a future decision on whether to simplify to the simple path.

---

## Design intent

- **Aptitudes** are intended to be measured **among themselves**: the 1-5 scale for aptitudes should reflect "strongest aptitude vs weakest aptitude" within the set of aptitudes only. That matches the idea that aptitudes are a different kind of construct (capacity / potential) and should be ranked on their own.
- **Traits and values** are intended to be measured **combined**: traits and values appear in the same triangles (e.g. a triangle with two values and one trait), so the 1-5 scale for traits/values is naturally relative to the joint set of traits and values.

---

## Data separation (Step 1)

- **dimension_triangles.json:** triangles use only `value_*` and `trait_*` dimension ids. No aptitudes.
- **dimension_triangles_aptitudes.json:** triangles use only `aptitude_*` dimension ids. No traits or values.
- **loadTriangles()** merges both files into one question list: `triangles = [...data1.triangles, ...data2.triangles]`. So the user sees one combined sequence of questions, but each triangle is either "all three vertices are traits/values" or "all three vertices are aptitudes." No triangle mixes an aptitude with a trait or value.

So at capture (Step 1) and per-triangle interpretation (Step 2), the separation is already in the data.

---

## Current implementation (Step 3): separate scaling implemented

The aggregation step **does** separate aptitudes from traits and values:

1. **collectTriangleRawResponses(state, answers)** returns one list of all triangle responses.
2. **splitRawResponsesByType(rawResponses)** splits into aptitude-only (all three dims start with `aptitude_`) and trait+value (the rest).
3. **scoreTriangleResponses** is called **twice**: once on the aptitude list, once on the trait+value list. Each run has its own θ vector and applies **thetaToScore15** within that run only (min/max over that group's dimensions). The two profiles are merged.
4. **Output:** One aptitude and one trait/value can both have mean 5.0 (each is strongest in its own group). So "5" means "strongest in this group," not "strongest overall."

**MCMC vs simple comparison:** The system also computes a **simple zone-weighted average** (zone weights: corner 1, near_corner 0.9, edge/near_edge 0.7, centre 0.3; then rescale to 1-5 within each group). It logs for each assessment: Spearman rank correlation between MCMC and simple order, whether top dimension agrees, whether bottom dimension agrees, and the dimension ids. This supports a future decision: if MCMC and simple agree often in practice, the codebase can be simplified to use only the simple path.

---

## Summary table

| Aspect | Implementation |
|--------|----------------|
| Triangle data (Step 1) | Two files; no mixed triangles |
| Step 3 input to scorer | Two lists (aptitude-only, trait+value-only) |
| Bayesian runs | Two (one per group) |
| Relative 1-5 scale | Over aptitudes only; over traits+values only |
| Interpretation | "Highest aptitude" and "Highest trait/value" each meaningful on their own |
| Comparison logging | MCMC vs simple zone-weighted rank order (Spearman, top/bottom agreement) logged per assessment |

---

## Possible future simplification

If comparison logs show that MCMC and simple zone-weighted average agree most of the time (e.g. same top/bottom, high Spearman), the production path could be switched to **simple zone-weighted average only**: remove the two MCMC calls and use `simpleZoneWeightedScores` output as the returned dimension scores. That would make Step 3 easier to explain and debug. The current implementation keeps MCMC as the returned scores and uses the simple path only for logging.

---

## Profile test interpretation (why "logical" can show all traits high)

The triangle tests include profile validation: "Highly logical," "Highly artistic," and "Super team player." **Only the logical profile** uses centre (1/3, 1/3, 1/3) for **every** value/trait triangle, because its target set is only `aptitude_logical_analytical_reasoning` (no value/trait dimensions). Artistic and team profiles target value_creativity_self_expression, value_helping_others_impact, trait_social_collaboration, etc., so in those triangles they submit **corners** (e.g. 1,0,0), which creates real differentiation and thus low scores for non-target dimensions. So the contrast is correct: logical has zero value/trait corners; the others have many.

When a group has no real differentiation (e.g. all centre), MCMC posterior thetas are similar; rescaling them min-to-1 and max-to-5 would force arbitrary extremes. The fix is **output-based**, not input-based: in the Bayesian scorer, when mapping theta means to the 1-5 scale we check the **range of thetas**. If the range is below a threshold (`THETA_SPREAD_NEUTRAL_THRESHOLD`, 0.5), we treat the group as undifferentiated and return **neutral** (3) for every dimension instead of rescaling. So we always run MCMC; the decision "no signal" is made from the posterior spread, not from counting centre responses or magic sample sizes. Result: the **logical** profile shows logical aptitude high and all traits/values 3; the **super team player** shows team dimensions elevated and all aptitudes 3. The test only asserts that **logical aptitude** is high and that team dimensions are elevated.

### Why one target dimension can still show "medium" (e.g. value_belonging_community 3.68)

Profile builders (e.g. the test's `buildProfileAnswers`) choose one vertex per triangle: when several target dimensions appear in the same triangle, they use **first matching vertex in order a, b, c**. So only one target gets the corner in that triangle; the others get weight 0 there. Example: **triangle_02** has vertices a = financial, b = work_life, c = belonging. For the super team player (targets: helping, belonging, collaboration, work_life), both **work_life** (b) and **belonging** (c) are targets; the first match is b, so work_life gets the corner and **belonging gets 0** in that triangle. Belonging is only maximized in **triangle_05** (one corner total). So it has less total signal than e.g. trait_social_collaboration (3 triangles). After MCMC and rescaling to [1, 5], belonging can land around 3.68 (medium) while collaboration is high. So a dimension that shares a triangle with another target and loses the "first match" can reasonably show medium rather than high.

## Low engagement (centre everywhere)

If a user leaves the ball in the middle for all or nearly all questions, the system receives undifferentiated centre data but still rescales to a full 1-5 profile, which can look arbitrarily strong or weak. That would contradict the intent that we encourage deliberate choices. The implementation therefore detects **low engagement** (e.g. fraction of centre responses above a threshold) and returns **neutral** dimension scores (mean 3, band medium) and a **lowEngagement** flag so the UI can show that choices are needed for a meaningful profile. See assessmentService and the `lowEngagement` field on the assessment response.

---

*This document is referenced from bft-doc/triangle-assessment-flow-for-expert-review.md (Step 3).*
