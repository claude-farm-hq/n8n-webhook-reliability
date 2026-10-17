# n8n webhook reliability: retry with backoff and a dead-letter queue

> **Status: tested headless on n8n 2.35.7 (self-hosted, local instance), 2026-10-13, by an AI agent working for Martin Minkov. Not yet imported through the editor UI by a human.**
> The test imported both workflows with the n8n CLI (after adding a top-level `id`, see "Import steps"), activated them and called the production webhooks against a local mock target. The tests below (success, transient failure, exhaustion, permanent 4xx, network error, dead-letter queue valid and invalid input, retry-to-dead-letter hand-off) behaved as described. Not tested: UI import and the canvas look, the Header Auth steps, a failing dead-letter queue call, other n8n versions and n8n Cloud. Still import into a test instance first and do not point them at production systems. Full record: TESTING.md; a summary is in "What was tested" below.

Two importable n8n workflows for webhook deliveries that sometimes fail.

- `workflows/retry-with-backoff.json` receives a JSON payload on a webhook and forwards it to a target URL you configure. It retries only transient failures (network error, HTTP 408, 429 and 5xx) with a delay that doubles on every attempt plus a little jitter, capped at 300 seconds. Other 4xx responses are treated as permanent and are not retried. After the last attempt it hands the payload to the dead-letter queue workflow.
- `workflows/dead-letter-queue.json` accepts a failed payload together with failure metadata, validates it, and stores one row per failure in an n8n Data Table with status `pending`, so it can be inspected later. Invalid input gets HTTP 400 and is not stored.

No credentials are embedded in either file and none are needed to import them.

Part of a planned set of six webhook reliability workflows; only these two exist.

Built and operated by Martin Minkov with AI agents. Contact: mminkov@aubian.net

The workflows and this README were written by an AI agent working for Martin Minkov, who owns and is accountable for them.

License: MIT (see LICENSE).

## What was tested, and what was not

Tested on 2026-10-13 on n8n 2.35.7, a local self-hosted instance (SQLite), driven headless through the CLI and REST API, with a local mock server as the target (max_attempts 3, base delay 2 s):

- Dead-letter queue: valid body gave HTTP 201 and one `pending` row; `{}` gave HTTP 400 and no row. The Data Table node parameters were accepted.
- Retry: success path one call; 503, 503, 200 gave waits of 2.33 s and 4.24 s; always 503 gave 3 calls (waits 2.32 s, 4.40 s) then a dead-letter row with `attempts=3, last_status=503`; 400 gave one call and a row with reason `non_retryable_status`; a refused connection gave 3 attempts and a row with the error text. In the three cases that ended in the dead-letter queue (always 503, 400, refused connection), the hand-off from the retry workflow to the dead-letter queue workflow worked (dead-letter webhook without Header Auth).
- `$runIndex` as attempt counter, the Wait node with an expression amount and the IF v2.2 conditions worked.
- `crypto.randomUUID` was not available in the Code node on this version: `dlq_id` uses the built-in timestamp-plus-random fallback.

Not tested, still unverified:

- Importing through the n8n editor (Import from File / paste) by a human, and how the canvas and sticky notes look. The n8n CLI `import:workflow` rejects these files because they carry no top-level `id`; add one (any unique string) to each file before using the CLI. The editor generates it itself, but that was not tested.
- The Header Auth hand-off steps under "Security: before activating", and the name and default of `N8N_PAYLOAD_SIZE_MAX` (recalled from memory, not checked against the n8n docs).
- A failing dead-letter queue call, HTTP 408 and 429, the 300 s cap, delays long enough to make n8n persist the waiting execution, the 15 s call timeout, volume, other n8n versions and n8n Cloud.
- Behaviour against any real endpoint.

If a node shows a red or "unknown parameter" warning after import, fix it by hand in the editor. Bug reports and corrections are welcome at the contact address above; no response time is promised.

## Security: before activating

Before activating, enable Header Auth on the Webhook node. These webhooks are unauthenticated by default. Create the credential in n8n; do not paste secrets into the JSON.

Payload size: the workflows themselves set no payload size limit; only your n8n instance's maximum request size applies. On self-hosted n8n that is an instance-wide setting (see `N8N_PAYLOAD_SIZE_MAX` in the n8n docs; check your own value, this README does not state one).

Dead-letter queue hand-off. `Send to Dead Letter Queue` in `workflows/retry-with-backoff.json` sends no auth header. If you protect the dead-letter queue webhook and change nothing else, that call is rejected, the retry execution fails and the failed payload can be lost. So when you protect the dead-letter queue webhook, also add the same header to the `Send to Dead Letter Queue` node:

1. Create the Header Auth credential in n8n (name and value of the header) and select it on the dead-letter queue workflow's `Webhook` node.
2. In the retry workflow, open `Send to Dead Letter Queue` and set its authentication to the generic credential type Header Auth, selecting that same credential. Use an n8n credential; do not paste the secret into the node's header fields or into the JSON. (Menu labels can differ between n8n versions.)
3. Test the exhaustion path again (see "Retry with backoff" below): the target always returns 503, and a row must appear in the Data Table. Also check that a call to the dead-letter queue webhook without the header is rejected.

The `curl` examples in this README send no header. Once Header Auth is on, add `-H` with your header name and value (from your shell, not typed into a saved file).

Each workflow has one main sticky note (yellow, top left) with an overview, setup steps and the owner line, plus white section notes that group the nodes. The tests are in this README and in TESTING.md.

## Import steps

1. In n8n: Workflows > top-right menu > Import from File (or paste the JSON onto the canvas).
2. Do not activate yet. (If you import with the n8n CLI instead of the editor, first add a top-level `"id": "<any unique string>"` to the JSON; the CLI fails with `NOT NULL constraint failed: workflow_entity.id` without it.)
3. Replace every `example.invalid` URL (the `Config` node in the retry workflow) with real URLs.
4. Delete the main (yellow) sticky note only after you have read it; it carries the owner line.
5. Credentials: none are embedded and none are needed to import. Header Auth credentials are added by you before activating; see "Security: before activating".

### Dead-letter queue first: `workflows/dead-letter-queue.json`

1. Create a Data Table, for example `dead_letters`, with string columns: `dlq_id, source, status, reason, attempts, last_status, last_error, payload_json, received_at`.
2. Open the `Store Dead Letter` node, select that table, and check the column mapping.
3. Set the workflow to test mode and copy the test URL.

Manual test. Put the test URL in a shell variable first:

```
WEBHOOK_URL='paste the test webhook URL here'
curl -X POST "$WEBHOOK_URL" -H "Content-Type: application/json" \
  -d '{"source":"manual-test","payload":{"order":1},"attempts":5,"reason":"max_attempts_reached"}'
```

Expected: HTTP 201, JSON `{"stored":true,"dlq_id":"..."}`, and one new row with status `pending`.

Invalid input test: send `{}` the same way. Expected: HTTP 400 and no new row.

### Retry with backoff: `workflows/retry-with-backoff.json`

1. In the `Config` node, set `target_url` to a URL you control and can make fail, and `dlq_url` to the dead-letter queue workflow's webhook URL (the production URL if the workflow is active, the test URL if it is listening).
2. Keep `max_attempts` small (3) and `base_delay_seconds` at 2 for testing.

Manual tests:

- Success path: the target returns 200. Expected: one call, and the execution ends at `Delivered`.
- Transient failure: the target returns 503 twice, then 200. Expected: delays of about 2s then 4s (plus jitter of up to 25% of the base delay), and the execution ends at `Delivered`.
- Exhaustion: the target always returns 503. Expected: 3 calls, then `Send to Dead Letter Queue` fires and a row appears in the Data Table. Run this again after you enable Header Auth on the dead-letter queue webhook and add the credential to `Send to Dead Letter Queue`.
- Permanent failure: the target returns 400. Expected: no retry, straight to the dead-letter queue with reason `non_retryable_status`.

## Known limits

These are by design for a minimal version:

- The retry webhook replies immediately (`onReceived`), so the caller does not learn the final outcome.
- The Wait node keeps executions open during delays, so long delays and high volume use execution storage.
- The delay is capped at 300 seconds.
- If the dead-letter queue call itself fails (a dead-letter queue outage), the execution fails and the payload can be lost. No `onError` fallback is built; watch your failed executions.
- Webhooks are unauthenticated by default; see "Security: before activating".
- The `Send to Dead Letter Queue` node sends no auth header. If you protect the dead-letter queue webhook, add the same header to that node using an n8n credential (not by pasting the secret into the JSON), then test the exhaustion path again. Otherwise the hand-off is rejected and the payload can be lost.
- The workflows set no payload size limit of their own; only your n8n instance's maximum request size applies.
