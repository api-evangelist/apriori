---
generated: '2026-08-06'
method: generated
name: Run a REST-driven costing job and collect results
description: Invoke an aP Connect workflow over an explicit list of PLM components with runPartList, poll the job to a terminal state, then pull per-part should-cost, DFM and carbon results.
api: openapi/apriori-ap-connect-agent.yml
operations: [listWorkflows, getWorkflow, runWorkflowAction, getWorkflowJob, getJobResults, getPartResults]
source: >-
  Grounded in openapi/apriori-ap-connect-agent.yml (OpenAPI 3.0.3, 12 operations), which API Evangelist
  transcribed from aPriori's published aP Connect Agent REST API Reference Guide at
  https://docs.apriori.com/en/Connect/apc/rarg/. Every operationId below is verified verbatim in that file.
  Auth per authentication/apriori-authentication.yml, async and error rules per
  conventions/apriori-conventions.yml and errors/apriori-problem-types.yml, entity graph per
  data-model/apriori-data-model.yml.
---

# Run a REST-driven costing job and collect results

The marquee flow of the aP Connect Agent REST API: hand aPriori a list of components you already have PLM identities for, let it cost them, and read back fully burdened cost, DFM risk and manufacturing carbon.

## Before you start

- **Base URL is `localhost`.** The aP Connect Agent runs on a customer-managed host inside your own network — aPriori documents the base URL as `localhost:<port_number>/`. There is no vendor-hosted origin. Your caller must be on a machine networked to the Agent.
- **The workflow must be configured for REST.** `Workflow.partSelectionType` must be `REST API request` for `runPartList` to be meaningful. Check it in step 2 before you invoke.
- aPriori states that an aPriori System Services engagement is recommended for leveraging this API on a file-system deployment.

## Auth

Send **either**:

- `Authorization: <JWT>` — the published "JWT Bearer" scheme (`apiKey`, in header), or
- `?key=<shared secret>` — the published "Shared Secret" scheme (`apiKey`, in query).

Prefer the header. A secret in the query string lands in every proxy and access log. Where the Connector is configured for mTLS (Agent 5.2.0+), the Agent host must also present its aPriori-signed certificate. See `authentication/apriori-authentication.yml`.

Set `Content-Type: application/json` on **every** request — `415 Unsupported Media Type` is documented on the GETs too.

## Steps

1. **Find the workflow** — `listWorkflows` (`GET /api/workflows`). Returns an array of `Workflow`. Pick the one you want and keep its `id`; that value is the `workflowIdentity` path parameter everywhere below.
2. **Confirm it is REST-driven and unlocked** — `getWorkflow` (`GET /api/workflows/{workflowIdentity}`). Check `partSelectionType == "REST API request"` and `locked == false`. A locked workflow is already executing.
3. **Invoke it** — `runWorkflowAction` (`POST /api/workflows/{workflowIdentity}/{action}`) with `action = runPartList`. Body:

   ```json
   {
     "parts": [
       {
         "id": "<your PLM part identity>",
         "costingInputs": {
           "processGroupName": "Plastic Molding",
           "materialName": "ABS",
           "annualVolume": "10000",
           "batchSize": "1000",
           "productionLife": "10",
           "vpeName": "aPriori USA",
           "scenarioName": "Initial"
         },
         "relativeCadFilePath": null
       }
     ]
   }
   ```

   **You must include the `costingInputs` object even if you set nothing in it** — that is aPriori's own published rule for `runPartList`. Any User Defined Attributes configured on the workflow can be passed as extra keys inside `costingInputs`.

   Use `action = run` instead when the workflow selects its own components from a file system or a query definition; that variant takes no body.

   The `200` response is a `WorkflowActionResult`. **Capture `jobId`.**

4. **Poll to terminal** — `getWorkflowJob` (`GET /api/workflows/{workflowIdentity}/jobs/{jobIdentity}`) with `jobIdentity = jobId`. Watch `status`, `completedAt`, and the counters `componentsTotal` / `componentsProcessed` / `componentsFailed`. The job is done when `completedAt` is populated and `status` is terminal. Back off between polls — aPriori publishes no `Retry-After` and no rate limits, so be a good citizen on a single-tenant Agent.
5. **Read all results** — `getJobResults` (`GET /api/workflows/{workflowIdentity}/jobs/{jobIdentity}/results`). Returns the job with per-part costing. Or read one component with `getPartResults` (`GET /api/workflows/{workflowIdentity}/jobs/{jobIdentity}/parts/{plmPartIdentity}/results`), where `plmPartIdentity` is the same PLM `id` you submitted in step 3.

## What comes back

Per part (`PartCostingResult`), the costing outputs live under `result` (`CostingResult`):

- **Cost** — `fullyBurdenedCost`, `totalCost`, `materialCost`, `capitalInvestment`
- **Process** — `cycleTime`, `laborTime`, `utilization`, `routingName`
- **Mass** — `roughMass`, `finishMass`
- **DFM** — `dfmRisk`, `dfmScore`
- **Carbon** — `totalCarbon`, `materialCarbon`, `processCarbon`, `logisticsCarbon`, `annualManufacturingCarbon`

and the assumptions it was costed under under `input` (`CostingInput`). Deep links back into aPriori come as `cidPartLink` and `apwScenarioLink`.

## Errors

- **`409 Conflict when job is not in terminal state`** on either results endpoint is the polling signal, not a failure. Go back to step 4. Do not tight-loop.
- **`400`** on step 3 usually means the body is missing `parts` or a part is missing `costingInputs`.
- **`401` / `403`** — no credential accepted, or the credential is not permitted for this Agent. Before mTLS shipped, `403` also covered a caller whose source IP was not on the Agent host allowlist.
- **`404`** — a stale `workflowIdentity` or `jobIdentity`. Re-resolve from step 1.
- There is **no error body**. Every non-2xx is documented with schema "No Content", so branch on the status code alone. See `errors/apriori-problem-types.yml`.
- Per-component failures do **not** produce an HTTP error — they arrive inside a `200` as `PartCostingResult.errorMessage` alongside `cicStatus`, and roll up into `WorkflowJob.componentsFailed` and `WorkflowJob.errorMessage`. Always check those before treating a `200` as success.

## Idempotency

There is none. No idempotency key is published anywhere in this API. If step 3 times out and you retry, you can start a **second** costing job. Record the `jobId` you got, and reconcile against `listWorkflowJobs` (`GET /api/workflows/{workflowIdentity}/jobs`) before re-invoking.

## Extra fields

Since Agent 4.0.0 the results payloads include **every User Defined Attribute** defined in your workflow setup. Treat `PartCostingResult` as an open object — the documented fields are a floor, not a ceiling.
