---
generated: '2026-08-06'
method: generated
name: Monitor an aP Connect Agent and its jobs
description: Read Agent health and configuration, inventory workflows, walk the job history, and cancel a stuck costing job.
api: openapi/apriori-ap-connect-agent.yml
operations: [getServiceStatus, getServiceConfiguration, listWorkflows, listWorkflowJobs, getWorkflowJob, runWorkflowJobAction]
source: >-
  Grounded in openapi/apriori-ap-connect-agent.yml (OpenAPI 3.0.3), transcribed from aPriori's published
  aP Connect Agent REST API Reference Guide at https://docs.apriori.com/en/Connect/apc/rarg/. Every
  operationId below is verified verbatim in that file. Conventions per
  conventions/apriori-conventions.yml; errors per errors/apriori-problem-types.yml.
---

# Monitor an aP Connect Agent and its jobs

The operational half of the aP Connect Agent REST API — is the Agent up, is it talking to the PLM, what is running, and how do I stop something that is stuck.

## Auth

`Authorization: <JWT>` header, or `?key=<shared secret>`. Prefer the header. Set `Content-Type: application/json` on every request, including GETs (`415` is documented on all of them). See `authentication/apriori-authentication.yml`.

## Steps

1. **Health check** — `getServiceStatus` (`GET /api/status`). Returns `ServiceStatus`:
   - `serviceStatus` — the Agent itself
   - `cicConnectionStatus` — link to aP Connect (Cost Insight Connect)
   - `plmConnectionStatus` — link to Windchill / Teamcenter / the file system
   - `jobCount` — how many jobs are in flight
   - `serviceTime` — Agent clock, format `yyyy-MM-dd'T'HH:mm'Z'`

   Both connection statuses matter independently. An Agent that is up but has lost its PLM connection will accept a workflow invocation and then fail the job.

2. **Read the configuration** — `getServiceConfiguration` (`GET /api/configuration`). Returns `ServiceConfiguration`. The fields worth alerting on:
   - `plmType` — one of `FILE_SYSTEM`, `MOCK`, `TEAMCENTER`, `WINDCHILL`. **`MOCK` in a production Agent is a misconfiguration.**
   - `cicHostUrl`, `fscUrl`, `hostname`, `rootFolderPath`
   - `scanRate`, `reconnectionInterval` — the polling and recovery cadence
   - `maxPartsToReturn` — the only volume ceiling this API publishes; results are truncated by it, not paginated
   - `csrfTokenTimeoutSeconds`

3. **Inventory the workflows** — `listWorkflows` (`GET /api/workflows`). Each `Workflow` carries `id`, `name`, `description`, `partSelectionType` (`Spreadsheet` | `REST API request` | `Query definition`) and `locked`. A workflow left `locked` with nothing running is worth investigating — aP Connect App 3.7.0 fixed a bug where a mapping to a nonexistent PLM attribute left a workflow permanently locked.

4. **Walk the job history** — `listWorkflowJobs` (`GET /api/workflows/{workflowIdentity}/jobs`). Returns `WorkflowJobSummary` with `identity`, `status` and `outputFolder`. There is **no pagination** — no page, cursor, limit or offset parameter exists — so expect the full list.

5. **Inspect one job** — `getWorkflowJob` (`GET /api/workflows/{workflowIdentity}/jobs/{jobIdentity}`). `WorkflowJob` adds `startedAt`, `completedAt`, `componentsTotal`, `componentsProcessed`, `componentsFailed` and `errorMessage`. Terminal = `completedAt` populated and `status` terminal.

6. **Cancel a job** — `runWorkflowJobAction` (`POST /api/workflows/{workflowIdentity}/jobs/{jobIdentity}/{action}`) with `action = cancel`. That is the only published action. Expect `201` or `202` — this is an accepted-for-processing response, not a completion, so re-poll step 5 to confirm the job actually reached a terminal state.

## Errors

- `401` / `403` — credential missing, rejected, or not permitted for this Agent.
- `404` — unknown `workflowIdentity` or `jobIdentity`; re-resolve from steps 3 and 4.
- `415` — send `Content-Type: application/json`.
- No error body is published for any status. See `errors/apriori-problem-types.yml`.

## Cautions

- **Cancel is a write.** Gate it behind explicit human confirmation in any agentic flow.
- Retrying a cancel is not idempotent in any published sense — there is no idempotency key on this API. Confirm state with step 5 rather than re-issuing.
