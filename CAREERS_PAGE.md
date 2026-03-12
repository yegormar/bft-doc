# Careers Page (Data-Driven)

The Careers page is **data-model-driven** and does **not** use an LLM for recommendations.

## Data flow

1. **Dimensions to skills**  
   Measured traits and values (from discovery) are mapped to skills via the existing model (`related_skill_clusters` in dimension data). The report’s `skillDevelopmentRoadmap` provides skills with an **applicability** score: fit to the user’s dimensions only (mean and band; no AI relevance in this number). So when the user gave no preference (all measurements 3), all skills that have at least one linked dimension get the same applicability. AI relevance (e.g. “Demand Grows”) is shown separately and is not part of applicability.

2. **Skills to time-investment buckets**  
   The user assigns skills to three buckets: **Low time investment**, **Medium time investment**, **High time investment** (high = “I will be studying hard”). Assignment is by drag-and-drop; there is no default by skill property.

3. **Selected skills and dimensions to careers**  
   The UI sends to the API: (1) **Skills:** for each skill in the three buckets: `id`, `bucket` (low/medium/high), and `applicability`. (2) **Dimension scores:** report's `dimensionScores` (traits and values with `id`, `mean`, `band`). Occupations are scored as a weighted combination of: **Skill match** (compatibilityRating * bucket weight * normalized applicability) and **Dimension (trait/value) match** (compatibilityRating * user dimension score). The split is configurable (default 60% skills, 40% dimensions). Each occupation also gets **aiRelevanceFromSkills** (0-1), the average of the AI future scores of the skills it maps to.

## Data sources

- **Skills and applicability:** Report API (core only, no `includeLlm`), i.e. `skillDevelopmentRoadmap` from `skillRecommendation.getSkillsWithApplicability(dimensionScores)`.
- **Occupations:** NOC data from `bft-api/src/data/noc-2021-enriched.json`. Each occupation has `skillMappings`, `traitMappings`, and `valueMappings`. All three are used when calling the match endpoint.
- **Config:** `OCCUPATION_SKILL_WEIGHT` and `OCCUPATION_DIMENSION_WEIGHT` in `.env` (must sum to 1). Default: 60% skills, 40% dimensions.

## API

- `POST /api/occupations/match` – Body: `{ skills: [{ id, bucket, applicability }], dimensionScores: { traits: [...], values: [...] }, groupBy?: 'category' }`. Returns `{ groups }` or flat list; each occupation has `matchScore` and `aiRelevanceFromSkills`. Used by the Careers page.
- `GET /api/occupations?skillIds=id1,id2,...` – Legacy: skill-ID-only scoring. Returns flat list or `{ groups }` with `groupBy=category`.
- `GET /api/occupations/:nocCode` – Full occupation including `aiRelevanceFromSkills` for the detail modal.

## UI behaviour

- **Top:** Skills as draggable tiles, coloured by applicability (red = low fit, yellow = medium, green = high fit; higher value = stronger colour).
- **Middle:** Three bucket sections; user drags skills from the pool into Low / Medium / High time investment.
- **Bottom:** Top 6 careers: first 3 are always the best match by score (so the user never misses their top fits); the next 3 are diversified (one per NOC major group not already in the top 3). Each card shows match score and AI relevance label. Click opens a modal with full occupation description and AI relevance.

## Optional LLM use (future)

The current flow is data-only. Possible LLM enhancements without replacing scoring:

- **Short rationale per career:** Given the 6 shown occupations and the user's selected skills, generate one sentence per career (e.g. "Fits your strategic planning and systems thinking focus."). Display under each card or in the detail modal.
- **Semantic clustering for diversity:** Instead of NOC major group, send occupation names (and optionally NOC descriptions) to an LLM and ask for 4–6 thematic clusters (e.g. "Leadership and policy", "Analysis and research"). Use cluster membership to pick diversified occupations by theme. Requires a small API that calls the LLM with the top N list and returns cluster labels and assignments.
- **One-line summary for the set:** e.g. "Your profile points to roles that combine strategy with people leadership." Could be generated when the Careers page loads (skills + top occupations) and shown above the cards.
