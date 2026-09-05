---
name: Audit and clean up Reckoner connections and accounts
description: Inventory a Reckoner project's saved service connections and member accounts through the External API, and remove stale ones — with the irreversibility warnings this API does not give you.
api: openapi/3shake-reckoner-external-api-openapi.yml
operations: [listProjects, getProject, listIntegrations, deleteIntegration, getProjectAccounts, deleteAccount, listWorkflows, getWorkflow]
generated: '2026-09-05'
method: generated
source: openapi/3shake-reckoner-external-api-openapi.yml
---

# Audit and clean up Reckoner connections and accounts

Base URL: `https://cdp-server.reckoner-api.com/api/external/v1`
Auth: `Authorization: Bearer <pat_…>`

**Read this first.** Two of the four write operations in this API are deletes, and **neither has a
reversal, a restore, or a documented retention window.** Nothing in the contract or the release notes
says a deleted account or connection can be recovered. Treat every delete in this skill as permanent
and gate it behind explicit human confirmation naming the exact id.

## Inventory (safe, read-only)

1. `listProjects` — `GET /projects`. No pagination.
2. `getProject` — `GET /projects/{projectId}`. Note `saves_job_result`: when it is `false`, job
   history is not retained, so an audit trail for this project does not exist.
3. `listIntegrations` — `GET /integrations/projects/{projectId}`, optionally filtered by `service`.
   The response is an object **keyed by service name**, each value an array of connection records
   (`integration_id`, `team_id`, `project_id`, `name`). The contract states that connection
   properties never include secrets, so this is safe to log. There is no pagination — the whole set
   comes back.
4. `getProjectAccounts` — `GET /projects/{projectId}/accounts`. Returns `id`, `email`,
   `is_team_admin`, and `roles` from the enum `project-admin`, `workflow-editor`,
   `integration-editor`, `read-only`. No pagination.
5. **Find what depends on a connection before touching it.** `listWorkflows`
   (`GET /workflows/projects/{projectId}`), then `getWorkflow` (`GET /workflows/{workflowId}`) for
   each candidate, and match `tasks[].integration_id` against the `integration_id` you are about to
   delete. There is no reverse lookup operation — this join is yours to do, and skipping it is how a
   cleanup breaks a scheduled overnight run.

## Deletes (irreversible — confirm each one)

- `deleteIntegration` — `DELETE /integrations/{serviceName}/{integrationId}`.
  The `force` query parameter deletes the connection **even when workflows still reference it**.
  `force` widens the blast radius; it does not make the operation safer. Prefer `force` absent, and
  only set it after step 5 has shown you exactly which workflows break.
- `deleteAccount` — `DELETE /accounts/{accountId}`, with a `cascade` query parameter.
  This is the highest-consequence operation in the API. Confirm the `email` from step 4 matches the
  person you intend to remove before sending the `id`; all ids here are bare integers with no type
  prefix, so an `accountId` and a `projectId` are indistinguishable by inspection.

## After any delete

Re-run the inventory steps to confirm the intended state. There is no undo, no soft-delete list, and
no restore operation to fall back on.

Conventions, reversibility notes and the full error table: `conventions/3shake-conventions.yml` and
`errors/3shake-problem-types.yml`.
