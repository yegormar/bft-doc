# Session export and restore (dev/debug)

Session export and restore is for **development and debugging only**. It is not part of the UI.

## Enable

Set in `.env`:

```bash
BFT_DEV_SESSION_EXPORT=1
```

If this variable is not set or not `1`, the export/restore endpoints are not registered.

## Export

Export the full state of a session (questions, answers, interview state, pregen queue).

- **Request:** `GET /api/dev/session/export/:sessionId`
- **Response:** `200` JSON object with:
  - `version` – export format version
  - `session` – session entity (id, status, createdAt, updatedAt, preSurveyProfile)
  - `answers` – all submitted answers
  - `assessmentSummaries` – Ollama assessment summaries per question
  - `interviewState` – askedQuestionIds, servedQuestions, coverage, dimensionScoresAggregate, etc.
  - `preGeneratedQueue` – pre-generated question queue

Example (save to file for later restore):

```bash
curl -s "http://localhost:3000/api/dev/session/export/YOUR_SESSION_ID" > session-backup.json
```

## Restore

Restore a session from a previously exported blob.

- **Request:** `POST /api/dev/session/restore`
- **Body:** JSON:
  - `{ "export": <exported blob> }` – overwrites the **original** session id (from the export) with the export data.
  - `{ "export": <exported blob>, "targetSessionId": "<ULID>" }` – overwrites the session with id `targetSessionId` with the export data. Use this to say which session to overwrite (e.g. the one currently open in the UI).
- **Response:** `201` with `{ "sessionId", "session" }`

Example (restore, overwriting the original session):

```bash
curl -s -X POST "http://localhost:3000/api/dev/session/restore" \
  -H "Content-Type: application/json" \
  -d "{\"export\": $(cat session-backup.json)}" \
  | jq
```

Example (restore, overwriting a specific session by id):

```bash
curl -s -X POST "http://localhost:3000/api/dev/session/restore" \
  -H "Content-Type: application/json" \
  -d "{\"export\": $(cat session-backup.json), \"targetSessionId\": \"SESSION_ID_TO_OVERWRITE\"}" \
  | jq
```

After restore, the report cache for that session is invalidated so the next report request recomputes from the restored data.
