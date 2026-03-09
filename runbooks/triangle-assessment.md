# Triangle assessment mode

When `BFT_ASSESSMENT_MODE=triangles`, the discovery flow uses the fixed triangle set from `bft-api/src/data/dimension_triangles.json` instead of LLM-generated scenario questions. No pregen or LLM is used.

## Enabling triangle mode

1. In **bft-api** `.env`, set:
   ```bash
   BFT_ASSESSMENT_MODE=triangles
   ```
2. Restart the API. When mode is `triangles`, pregen and interview min-signal config are not required (they are only used when `BFT_ASSESSMENT_MODE=scenarios`).

## Flow

- User starts discovery as usual (pre-survey then main survey).
- Each "question" is one triangle: title, prompt, and three vertices (A, B, C) with labels.
- User drags a ball on the triangle to indicate their position; the position is sent as barycentric coordinates `{ a, b, c }` (sum = 1).
- Scores are computed per vertex: `score = (coordinate * 4) + 1` (mapping [0,1] to [1,5]). When a dimension appears in multiple triangles, scores are averaged.
- Completion: after all triangles in the JSON have been served, the interview is complete and the user can view results.

## Methodology

See **bft-doc/triangle_assessment_methodology.md** for the assessment design, scoring, and when to use triangles vs scenarios.

## Re-enabling scenario mode

Set in `.env`:

```bash
BFT_ASSESSMENT_MODE=scenarios
```

Then ensure pregen and interview config are set (e.g. `BFT_PREGEN_QUEUE_CAP`, `BFT_PREGEN_REFILL_THRESHOLD`, `MIN_SIGNAL_PER_DIMENSION`). Restart the API.
