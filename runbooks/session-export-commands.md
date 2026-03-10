# Session export/restore – quick commands

Dev/debug only. Full description: [session-export-restore.md](./session-export-restore.md).

## Enable

In `.env`:

```
BFT_DEV_SESSION_EXPORT=1
```

Restart the API after changing.

---

## Export (save session to file)

Replace `YOUR_SESSION_ID` with the real session ULID (e.g. from the browser URL).

```bash
curl -s "http://localhost:3000/api/dev/session/export/YOUR_SESSION_ID" > session-backup.json
```

---

## Restore (overwrite the original session)

Puts the export back under the same session id that was exported. That session is overwritten.

```bash
curl -s -X POST "http://localhost:3000/api/dev/session/restore" \
  -H "Content-Type: application/json" \
  -d "{\"export\": $(cat session-backup.json)}" \
  | jq
```

---

## Restore and overwrite a specific session

Use this when you want to **overwrite** an existing session (e.g. the one currently in the UI) with the exported data. Set `targetSessionId` to that session’s id; its data will be replaced by the export.

Replace `SESSION_ID_TO_OVERWRITE` with the session id you want to overwrite (e.g. from the browser URL or the session you just finished).

```bash
curl -s -X POST "http://localhost:3000/api/dev/session/restore" \
  -H "Content-Type: application/json" \
  -d "{\"export\": $(cat session-backup.json), \"targetSessionId\": \"SESSION_ID_TO_OVERWRITE\"}" \
  | jq
```

---

## After restore: open the restored session in the UI

The app uses **sessionStorage** for the "current" session. If you had just finished a different session, the UI is still showing that one. To see the restored data you must open the **restored** session id.

Open the results page with the restored session id in the URL (replace with your base URL and session id):

```
http://localhost:5173/results?sessionId=01KKACN2CG43WE003132HZVQ07
```

That loads the restored session and updates sessionStorage, so all result sub-pages (Your answers, Traits and Values, etc.) will show the restored data. The report cache for that session is cleared on restore so the next load recomputes from the restored data.
