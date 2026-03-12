# Skills UI: display only, no scaling

The UI displays the numbers from the API. **All calculations are done on the API**; the UI does not scale or normalize applicability.

---

## Two different numbers


| Source                              | API field         | Meaning                                                                                                           | API range                           |
| ----------------------------------- | ----------------- | ----------------------------------------------------------------------------------------------------------------- | ----------------------------------- |
| **User skills** (your match)        | `applicability`   | Fit from dimension scores only: weighted average of dimension means (1-5) by dimension_skill_mapping.json. Band is for labels only, not used in calculation. | 1 to 5 (or 0 when no link).         |
| **AI skills** (demand / future fit) | `ai_future_score` | How relevant or resilient the skill is in the AI era (from structural_scores in skills.json; not user-dependent). | **0–1** (fixed).                    |


---

## User skills (applicability): API only, UI displays as-is

**API:** Skill calculation is based only on **dimension_skill_mapping.json**. For each skill, `applicability = weighted average of dimension means (1-5)` using the mapping weights. Band is **not** used in the calculation; band is for display (e.g. "High" / "Medium" / "Low" label on dimensions). Result is clamped to 1-5; 0 when no dimension links to the skill.

**UI (list and detail):** The UI shows the API value. No scaling, no "best = 5" normalization, no min/max spread logic. Display = applicability clamped to 1-5 for safety; band label comes from config (bandsRanges) or a fixed 1-5 threshold. Example: if API returns applicability 2.1, the UI shows 2.1/5.

---

## AI skills (ai_future_score → displayed 1–5)

**API:** `ai_future_score` is **0–1**. It comes from `computeAiFutureScore(skill)`: each of the six structural dimensions (e.g. ai_resistance, leverage_multiplier) is 0–5 in skills.json; we convert to 0–1 (divide by 5), invert when the model says so, then average. Result is always in **[0, 1]**. Same for every user (not based on assessment).

**UI:** We map **0–1 to 1–5** with a **fixed** formula:

- display = 1 + ai_future_score × 4

So 0 → 1, 0.5 → 3, 1 → 5. There is **no** "normalize by max" here. The scale is **absolute**: 0.9 always shows as 4.6, no matter what other skills are in the list.

**Examples (AI skills):**


| ai_future_score (API) | Displayed score | Band (example) |
| --------------------- | --------------- | -------------- |
| 0.2                   | 1.8             | Low            |
| 0.5                   | 3.0             | Medium         |
| 0.9                   | 4.6             | Very High      |
| 1.0                   | 5.0             | Very High      |


So **AI skill** display is **absolute** (same 0–1 → 1–5 mapping every time).

---

## Radar chart (Skills page)

- **Your match (blue):** Uses API applicability directly (clamped 1-5). No scaling.
- **AI future fit (red):** `ai_future_score` (0–1) from API → display = 1 + ai_future_score×4 (fixed scale 1-5).

---

## Summary


|                   | User skills (applicability)                           | AI skills (ai_future_score)                         |
| ----------------- | ----------------------------------------------------- | --------------------------------------------------- |
| **API range**     | 1 to 5 (or 0 when no link)                             | **0–1** (fixed)                                     |
| **UI**            | Display API value only (no scaling)                    | display = 1 + (0–1)×4                               |
| **Depends on**    | This user's dimension scores                           | Skill's structural_scores only (same for all users)  |


