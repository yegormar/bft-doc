# Careers Page (Data-Driven)

The Careers page is **data-model-driven** and does **not** use an LLM for recommendations.

## Data flow

1. **Dimensions to skills**  
   Measured traits and values (from discovery) are mapped to skills via the existing model (`related_skill_clusters` in dimension data). The report’s `skillDevelopmentRoadmap` provides skills with an **applicability** score (match to the user’s dimensions).

2. **Skills to time-investment buckets**  
   The user assigns skills to three buckets: **Low time investment**, **Medium time investment**, **High time investment** (high = “I will be studying hard”). Assignment is by drag-and-drop; there is no default by skill property.

3. **Selected skills to careers**  
   The union of skill IDs from all three buckets is sent to the API. Occupations (NOC) are scored by how well their `skillMappings` match those IDs (sum of `compatibilityRating`). Results are sorted by match score.

## Data sources

- **Skills and applicability:** Report API (core only, no `includeLlm`), i.e. `skillDevelopmentRoadmap` from `skillRecommendation.getSkillsWithApplicability(dimensionScores)`.
- **Occupations:** NOC data from `bft-api/src/data/noc-2021-enriched.json`. Each occupation has `skillMappings`, `traitMappings`, and `valueMappings` with compatibility ratings. The Careers page uses `skillMappings` for scoring.

## API

- `GET /api/occupations?skillIds=id1,id2,...` – returns a flat list of `{ nocCode, name, matchScore, categoryKey, categoryLabel }` sorted by match score.
- `GET /api/occupations?skillIds=...&groupBy=category` – returns `{ groups: [ { categoryKey, categoryLabel, occupations } ] }` with groups ordered by best match in category; occupations within each group sorted by match score. Categories use NOC 2021 major groups (first 2 digits of nocCode); labels come from `bft-api/src/data/noc-2021-major-groups.json`.
- `GET /api/occupations/:nocCode` – returns the full occupation (name, exampleTitles, mainDuties, employmentRequirements, additionalInformation) for the detail modal.

## UI behaviour

- **Top:** Skills as draggable tiles, coloured by applicability (red = low fit, yellow = medium, green = high fit; higher value = stronger colour).
- **Middle:** Three bucket sections; user drags skills from the pool into Low / Medium / High time investment.
- **Bottom:** Matching NOC occupations grouped by **category** (NOC 2021 major group, e.g. "Professional in finance and business", "Technical in natural and applied sciences"). Categories are ordered by best match in that group. Within each category, occupations are listed by match score; click opens a modal with full occupation description (same pattern as Skills and Traits pages).
