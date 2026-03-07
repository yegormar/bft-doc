# Scenario prepopulation (admin tool)

This runbook describes how to use the scenario prepopulation runner and admin API to fill the question store with generated scenarios before testing.

## Prerequisites

- bft-api `.env` configured (see env.example), including:
  - `BFT_QUESTIONS_STORE_DIR` (required)
  - `MIN_SIGNAL_PER_DIMENSION` (required for interview config)
  - LLM and scenario prompt paths
- Run commands from the **bft-api** directory so relative store path resolves correctly.

## Prepopulation runner

The runner generates scenarios for synthetic pre-test profiles until every dimension (aptitudes, traits, values, skills) has at least `MIN_SIGNAL_PER_DIMENSION` questions per profile, then moves to the next profile. It can run indefinitely to build hundreds of scenarios.

### Run from bft-api

```bash
cd bft-api
node scripts/scenario-pregen.js
```

- Runs until you press Ctrl+C. After each profile completes, it rotates to the next; when all profiles are done, it starts a new cycle.
- Logs: per-profile count, total in store every 5 questions, and final store total when stopped.

### Options

- `--once`: Run one pass over all profiles then exit (no repeat cycles).
- `--profiles N`: Run at most N full profile cycles then exit.

Example (one cycle then exit):

```bash
node scripts/scenario-pregen.js --once
```

Example (two full cycles then exit):

```bash
node scripts/scenario-pregen.js --profiles 2
```

### Output

- Initial store total.
- For each profile: `profile complete profileIndex=... profileKey=... generated=...`
- After each profile: `total scenarios in store=...`
- On exit: `done. totalScenarios=...`

## Admin API

When `BFT_ADMIN_ENABLED=1` in bft-api `.env`, the API mounts admin routes under `/api/admin`. Use the **bft-api** server port (e.g. 3000), not the UI dev server (e.g. 5173).

Base URL: `http://localhost:PORT/api/admin` (PORT is from bft-api `.env`, e.g. 3000). There is no response at `/api/admin` itself; use the paths below.

### GET /api/admin/scenarios/stats

Returns store statistics. Use this URL to confirm the admin API is reachable.

- `totalScenarios`: total count
- `byProfileKey`: `{ [profileKey]: count }`
- `byDimension`: `{ [dimensionType]: { [dimensionId]: count } }`

### GET /api/admin/scenarios/profile-keys

Returns `{ profileKeys: string[] }` (list of profile keys).

### GET /api/admin/scenarios

List scenarios with optional query params:

- `profileKey`: filter by profile key
- `dimensionType`: filter by dimension type (aptitude, trait, value, skill)
- `dimensionId`: filter by dimension id
- `createdAfter`: ISO date string
- `createdBefore`: ISO date string
- `limit`: max number to return

Response: `{ scenarios: [...], count: N }`. Each scenario has `profileKey`, `fileName`, `createdAt`, `contentHash`, `question`, `dimensionSet`, `assessmentSummary`.

### DELETE /api/admin/scenarios/:profileKey/:contentHash

Deletes one stored scenario. `contentHash` can be the full hash or the first 16 characters (as in the filename). Returns `{ deleted: true }` or 404 with `{ deleted: false, message }`.

## Admin UI (optional)

A minimal HTML UI is provided at `bft-doc/admin/scenario-admin.html`. Open it in a browser and set the API base URL (e.g. `http://localhost:3001`) to:

- View stats (total, by profile, by dimension)
- List scenarios with filters (profile, dimension type/id, limit)
- Delete a scenario by profile key and content hash

If the API is on another origin, CORS must allow it (same as main app `CORS_ORIGIN` or add the page origin).

## Flow summary

1. Start the prepopulation runner from bft-api and let it run (hours if desired). It uses the same LLM and store as the product.
2. Optionally enable admin (`BFT_ADMIN_ENABLED=1`), start the API, and use the admin UI or API to view counts and list scenarios.
3. Delete any bad scenarios via the admin API or UI.
4. During real testing, the product uses the same store for fallback when LLM is slow or unavailable.
