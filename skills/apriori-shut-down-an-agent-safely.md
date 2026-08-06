---
generated: '2026-08-06'
method: generated
name: Shut down an aP Connect Agent safely
description: Use aPriori's two-step nonce handshake to drain and terminate an aP Connect Agent without abandoning in-flight costing jobs.
api: openapi/apriori-ap-connect-agent.yml
operations: [getServiceStatus, createShutdownNonce, initiateShutdown]
source: >-
  Grounded in openapi/apriori-ap-connect-agent.yml (OpenAPI 3.0.3), transcribed from aPriori's published
  aP Connect Agent REST API Reference Guide at https://docs.apriori.com/en/Connect/apc/rarg/. Every
  operationId below is verified verbatim in that file. See conventions/apriori-conventions.yml for the
  replay-safety discussion.
---

# Shut down an aP Connect Agent safely

**DESTRUCTIVE.** This flow stops a production integration between a PLM system and aPriori. Require explicit human confirmation before running it.

aPriori guards shutdown with a two-step nonce handshake — the only replay control published anywhere in this API. You ask the Agent for a single-use code, then present that code back to authorise the shutdown.

## Auth

`Authorization: <JWT>` header, or `?key=<shared secret>`. Set `Content-Type: application/json`. See `authentication/apriori-authentication.yml`.

## Steps

1. **Check what you are about to stop** — `getServiceStatus` (`GET /api/status`). Read `jobCount`. If jobs are in flight, decide deliberately: the only published shutdown mode drains them (see step 3).
2. **Mint the nonce** — `createShutdownNonce` (`POST /api/shutdown`). The `200` response is a `Shutdown` object carrying `shutdownCode`. It is single-use.
3. **Initiate the shutdown** — `initiateShutdown` with the nonce as the path segment and the mode in the body:

   ```json
   { "shutdownMode": "completeAndTerminate" }
   ```

   `completeAndTerminate` is the **only** mode aPriori publishes: the Agent finishes what it is already running, then terminates. Expect `204 Shutdown initiated` (or `201`).

   > **Path caveat.** aPriori's reference page renders this operation's request line as `POST /api/shutdown` while simultaneously documenting `nonce` as a **required path parameter**. Two operations cannot share one method+path, so `openapi/apriori-ap-connect-agent.yml` templates it as `POST /api/shutdown/{nonce}` and flags it `x-inferred: true`. Confirm the exact path against your own Agent's live spec at `http://localhost:<port_number>/v4/api-docs` before automating this step.

4. **Verify** — `getServiceStatus` stops answering once the Agent terminates. Treat a connection failure here as the success signal, not an error.

## Errors

- `400` — malformed body or a mode other than `completeAndTerminate`.
- `401` / `403` — credential rejected or not permitted.
- `404` — the nonce is unknown, already used, or expired. Go back to step 2 and mint a fresh one.
- `415` — send `Content-Type: application/json`.
- No error body is published. See `errors/apriori-problem-types.yml`.

## Why the nonce matters

This handshake is a **confirmation** mechanism, not general request idempotency — the rest of this API has none. Because `createShutdownNonce` mints a fresh code every call, calling it repeatedly is harmless; only `initiateShutdown` is destructive, and it will only accept a code once.
