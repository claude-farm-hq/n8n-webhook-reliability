# Test record: n8n 2.35.7, 2026-10-13

Built and operated by Martin Minkov with AI agents. Contact: mminkov@aubian.net

This test was run, and this record was written, by an AI agent working for Martin Minkov, who owns this repository and is accountable for it. The test was headless (command line and REST API against a throwaway local n8n instance). No human imported the files through the n8n editor UI; that remains untested. Martin Minkov accepted this test run on 2026-10-14 as the pre-publication test of these two workflows.

Everything below, except the file comparison in "Files tested vs files in this repo", was measured in that one run. Where something was not run, it is listed under "Not tested".

## Setup

```
node -v                       # v22.22.0
npm init -y && npm install n8n
./node_modules/.bin/n8n --version   # 2.35.7
export N8N_USER_FOLDER=$PWD/home N8N_DIAGNOSTICS_ENABLED=false N8N_VERSION_NOTIFICATIONS_ENABLED=false N8N_LISTEN_ADDRESS=127.0.0.1
```

- n8n version: 2.35.7 (npm package `n8n`, self-hosted, SQLite, no cloud, no n8n.io account). Node v22.22.0.
- n8n asked for an owner account via `POST /rest/owner/setup`. A throwaway local owner login (`test@example.invalid`) was created on the local instance, because the REST API for Data Tables and for activating workflows needs a session. No real address was used, and the login exists only in the throwaway instance.
- Target API (mock): a 25-line Node HTTP server on `127.0.0.1:9000`. Its mode (`ok`, `flaky` = 503 twice then 200, `always503`, `400`) is switched by a `GET` to `/_mode/` followed by the mode name, and it logs a timestamp per call. Its code is not part of this repository.
- Data Table `dead_letters` was created via `POST /rest/projects/PROJECT_ID/data-tables` with the nine string columns listed in the README.

## Import

`n8n import:workflow --input=FILE` on the unmodified repo files fails for both:

```
SQLITE_CONSTRAINT: NOT NULL constraint failed: workflow_entity.id
```

Cause: the JSON files have no top-level `id`. The CLI import requires one; the editor's Import from File or paste generates it itself (not tested here, UI import is untested). For the test, copies with an added `id` (`dlqwf000000000001`, `retrywf00000000001`) were written outside the repository and imported instead: "Successfully imported 1 workflow." for each. Nothing else in the files was changed. The files in this repository were left without `id`, as in the n8n export format for sharing; the README states that the CLI needs one added.

Node `typeVersion`s all loaded without error in 2.35.7 (Webhook 2, Set 3.4, HTTP Request 4.2, IF 2.2, Code 2, Wait 1.1, Respond to Webhook 1.1, Data Table 1). The imported workflows were then edited through `PATCH /rest/workflows/ID`: Data Table selected in `Store Dead Letter` (mode `id`), and `Config` set to `target_url=http://127.0.0.1:9000/target`, `dlq_url=http://localhost:5678/webhook/dead-letter-queue`, `max_attempts=3` (`base_delay_seconds` left at 2). Both were activated through `POST /rest/workflows/ID/activate`. Tests then called the production webhook URLs (`/webhook/...`), not test URLs.

## Files tested vs files in this repo

The test ran on the workflow JSON files as they were before the test, plus the added `id` described above. The two workflow files in this repository differ from those versions only in the text of the sticky note (the node named `Note` in each file), which was changed to state the test result. No node, parameter or connection changed.

This was checked on 2026-10-14 by parsing both versions of each file as JSON and comparing them deep-equal with the `parameters.content` of the `Note` node set aside; the rest of each file (nodes, parameters, connections, positions) was identical.

## Dead-letter queue results

```
curl -X POST localhost:5678/webhook/dead-letter-queue -H "Content-Type: application/json" \
  -d '{"source":"manual-test","payload":{"order":1},"attempts":5,"reason":"max_attempts_reached"}'
-> {"stored":true,"dlq_id":"1791933523426-59c4b50c5bb37"}   HTTP 201

curl ... -d '{}'
-> {"stored":false,"error":"body.payload is required"}      HTTP 400
```

Data Table after both calls: exactly one row (`source=manual-test`, `status=pending`, `reason=max_attempts_reached`, `attempts="5"`, `payload_json="{\"order\":1}"`, ISO `received_at`). The `{}` call stored nothing. **PASS**, matches the README (201 and one pending row; 400 and no row).

Observed: `dlq_id` is the timestamp fallback (milliseconds, a dash, then hex), so `crypto.randomUUID` was not available in the Code node sandbox in 2.35.7. The fallback works. The Data Table node parameters (`resource: row`, `operation: insert`, `dataTableId`, `columns` defineBelow) were accepted and wrote rows.

## Retry-with-backoff results

Four requests to `POST /webhook/retry-with-backoff` were recorded, each answered `{"message":"Workflow was started"}` HTTP 200 (n8n's default immediate acknowledgement). Executions were listed via `GET /rest/executions`; all `success`. For the network-error test no request/response line was recorded; its result is in the table.

| Test | Target log (UTC, from mock) | Observed | Result |
|---|---|---|---|
| Success (`ok`) | 1 call at 23:18:49.80 -> 200 | 1 call; execution 3 success, ~0.07 s | PASS |
| Transient (`flaky`: 503, 503, 200) | calls at 23:18:52.82, :55.15, :59.39 | gaps 2.33 s then 4.24 s (base 2 s / 4 s + jitter up to 0.5 s); execution 4 success; no DLQ row | PASS |
| Exhaustion (`always503`, max 3) | calls at 23:19:04.86, :07.18, :11.59 | gaps 2.32 s, 4.40 s; 3 calls then DLQ execution 6; row `source=retry-with-backoff, reason=max_attempts_reached, attempts=3, last_status=503` | PASS |
| Permanent (`400`) | 1 call at 23:19:20.92 | no retry; DLQ execution 8; row `reason=non_retryable_status, attempts=1, last_status=400` | PASS |
| Network error (target process stopped) | connection refused | 3 attempts over ~7 s; row `reason=max_attempts_reached, attempts=3, last_status="", last_error="connect ECONNREFUSED 127.0.0.1:9000"` | PASS |

Hand-off from the retry workflow to the dead-letter queue (payloads such as `{"order":"always503"}` stored as `payload_json`) works end to end with an unprotected dead-letter queue webhook. `$runIndex` as attempt counter, the Wait node with an expression amount, the IF v2.2 condition format and `neverError` with `onError: continueRegularOutput` for the network-error path all behaved as intended.

## Not tested (stay unverified)

- Import through the editor UI by a human (Import from File or paste). Only CLI import of copies with an added `id` was done.
- Sticky note rendering and box size (`height`/`width`), canvas layout, node positions: UI-only, not testable headless.
- Header Auth steps ("Security: before activating" in the README): no credential was created, and the hand-off with a protected dead-letter queue webhook and the "rejected without header" check were not run.
- Behaviour when the dead-letter queue call itself fails (a documented limit), the 408 and 429 status paths, the 300 s cap, the `max_attempts=5` default, Wait over long delays (n8n persists waiting executions to the database beyond a threshold; only 2 to 5 s waits were run), the 15 s timeout.
- The `N8N_PAYLOAD_SIZE_MAX` name and default (the README says it was recalled from memory).
- Other n8n versions, n8n Cloud, other Data Table feature availability, concurrency and volume.
- The test-URL (test mode) flow; only production URLs of active workflows were used.

## Bugs found

None in the workflow logic. No edits to the workflow behaviour were needed. The only finding is the missing top-level `id` for CLI import, documented in the README rather than changed in the JSON.

## Cleanup

n8n and the mock server were stopped afterwards (`ps` showed neither). The scratch directory (database, test owner login, `node_modules`) was outside any repository and is not part of this repository.
