---
name: Run a Reckoner workflow and wait for the result
description: Start a Reckoner data-integration workflow through the External API, then poll the job to a terminal state and read any errors — with the retry and cancellation rules this API actually requires.
api: openapi/3shake-reckoner-external-api-openapi.yml
operations: [listProjects, listWorkflows, runWorkflow, getWorkflowJob, cancelWorkflowJobs, authTokenRefresh]
generated: '2026-09-05'
method: generated
source: openapi/3shake-reckoner-external-api-openapi.yml
---

# Run a Reckoner workflow and wait for the result

Base URL: `https://cdp-server.reckoner-api.com/api/external/v1`

## Before you start

- Authenticate with `Authorization: Bearer <pat_…>` — the access token. Do **not** send the `prt_…`
  refresh token on business operations; it is accepted only by `authTokenRefresh`.
- **This API has no idempotency key.** There is no `Idempotency-Key` header anywhere in the contract.
  If `runWorkflow` times out or returns 500, do **not** blindly retry — a second call starts a
  second job that moves data again. Recover by listing recent jobs instead (see step 5).
- There is no dry-run or preview parameter. A `runWorkflow` call is the real thing.

## Steps

1. **Find the project.** `listProjects` — `GET /projects`. No parameters, no pagination. Take the
   `id` of the project you want; `is_default: true` marks the default one.

2. **Find the workflow.** `listWorkflows` — `GET /workflows/projects/{projectId}`. Narrow with
   `search_words` (partial match on the workflow name), `label_ids` (comma-separated), and
   `schedule_filtering_by` (`active` | `inactive` | `all`). Page with `limit` (max **100**, default
   100) and `page` (1-indexed); read `total` from the envelope to know whether more pages exist.
   Sort with `sort` in the form `<field>:<direction>` where field is `name`, `updated_at` or
   `last_started_at`.

3. **Start the run.** `runWorkflow` — `POST /workflows/{workflowId}/run`. The request body is
   optional; when supplied it is a `WorkflowRunParameter` that **overrides the workflow's parameter
   variables for this run only**. A 200 returns a `JobId`. Keep that id — it is the only handle you
   have on the run.

4. **Poll to a terminal state.** `getWorkflowJob` — `GET /workflows/{workflowId}/jobs/{jobId}`.
   Read `status`, not the HTTP status: a failed job returns **HTTP 200** with
   `status: FAILED` (or `SERVER_ERROR`) and an `errors[]` array of `WorkflowJobError`.

   | status | meaning | keep polling? |
   |---|---|---|
   | `SUBMITTING` | job being submitted | yes |
   | `RUNNABLE` | queued | yes |
   | `RUNNING` | executing | yes |
   | `CANCEL_STARTED` | cancellation in flight | yes |
   | `COMPLETED` | succeeded | no |
   | `FAILED` | failed — read `errors[]` | no |
   | `CANCELED` | cancelled | no |
   | `SERVER_ERROR` | platform failure | no |

   Back off between polls. The API returns **429** with
   `{"code":"RATE_LIMIT_EXCEEDED"}` when you go too fast, and publishes **no** `Retry-After` and no
   `RateLimit-*` headers, so choose your own exponential backoff and treat 429 as "wait longer",
   never as "the job failed".

5. **If step 3 timed out and you do not know whether the job started**, call `listWorkflowJobs` —
   `GET /workflows/{workflowId}/jobs` with `started_at` set to today (dates are **JST**) — and look
   for a job whose `trigger` is `external_api`. That is the safe alternative to re-running.

6. **To stop a run**, `cancelWorkflowJobs` — `PUT /workflows/{workflowId}/jobs/{jobId}/cancel`. This
   is the only reversal path in the API. It is meaningful while the job is `SUBMITTING`, `RUNNABLE`
   or `RUNNING`; the contract does not state what it does against a terminal job, and **cancelling
   does not roll back rows a sink task has already written downstream.** No cancellation window is
   published — do not assume one.

## Errors you must handle differently

| Status | `code` | What to do |
|---|---|---|
| 401 | `TOKEN_EXPIRED` | Call `authTokenRefresh` (`POST /auth/token/refresh`) with the `prt_` refresh token, take the new `access_token`, retry once. |
| 401 | `TOKEN_INVALID`, `TOKEN_REVOKED`, `UNAUTHORIZED` | Stop. A human must issue a new token in the console. Retrying never helps. |
| 402 | `SUBSCRIPTION_INACTIVE` | Stop and escalate. Commercial state, not a request defect. |
| 403 | `FORBIDDEN` | The token's account lacks the role. Roles are `project-admin`, `workflow-editor`, `integration-editor`, `read-only`. |
| 403 | `FEATURE_NOT_AVAILABLE` | The tenant's plan does not include the feature — a plan boundary, not a permission one. |
| 404 | `NOT_FOUND` | Check the `workflowId` / `jobId`. All ids are bare integers with no type prefix, so it is easy to pass the wrong one. |
| 429 | `RATE_LIMIT_EXCEEDED` | Back off and retry. No `Retry-After` is sent. |
| 500 | `INTERNAL_SERVER_ERROR` | Retry reads freely. For `runWorkflow`, go to step 5 before retrying. |

Every error body is `{"code": "...", "message": "..."}` served as `application/json` — this API is
**not** RFC 9457. Full catalogue: `errors/3shake-problem-types.yml`.
