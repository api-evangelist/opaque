---
name: Run a confidential analytics job and collect its results
description: Create a job in a workspace, attach its query and inputs, take it through review, run it, and fetch results and logs.
api: openapi/opaque-platform-api-openapi.yml
operations: [get_workspaces, get_data_by_workspace, create_job, update_job_query, get_job_query, update_job_predefined_query, create_job_input_variable, update_job_input_variable, delete_job_input_variable, update_job, review_job, get_job_reviews, create_job_run, get_job, get_jobs_by_workspace, get_job_run_results, get_job_run_logs, cancel_job_run]
generated: '2026-08-04'
method: generated
source: openapi/opaque-platform-api-openapi.yml
---

# Run a confidential analytics job and collect its results

A job is a query that executes inside a trusted execution environment against data shared into a
workspace. Jobs are **approval-gated**: nothing runs until the workspace's reviewers accept it.

Authenticate first — see `opaque-authenticate.md`.

## Steps

1. **Pick the workspace.** `get_workspaces` (`GET /{version}/workspaces`) lists them. Use one whose
   `type` is `analytics` and whose `status` is `verified`. Confirm the data you need is there with
   `get_data_by_workspace` (`GET /{version}/workspace/{workspace-uuid}/data`).
2. **Create the job.** `create_job` (`POST /{version}/workspace/{workspace-uuid}/job`). The job is
   created in `draft`. `400` on this call means an invalid body parameter.
3. **Attach the query.**
   - Python query: `update_job_query` (`PUT /{version}/job/{job-uuid}/query`). The body carries a
     **base64-encoded protobuf** definition. `400` means the definition is not valid base64 or
     cannot be parsed as a protobuf. Read it back with `get_job_query`.
   - Predefined query: `update_job_predefined_query`
     (`PUT /{version}/job/{job-uuid}/predefined-query`). Find the available templates with
     `get_available_predefined_query_templates`, `get_workspace_predefined_query_templates`, or
     `get_organization_predefined_query_templates`.
4. **Add input variables** if the job is parameterized. `create_job_input_variable`
   (`POST /{version}/job/{job-uuid}/input-variable`); amend with `update_job_input_variable`,
   remove with `delete_job_input_variable`. All three require the job to still be in `draft` —
   `400` otherwise.
5. **Send it to review.** Move the job to `under review` with `update_job`
   (`PATCH /{version}/job/{job-uuid}`), then have a reviewer call `review_job`
   (`PATCH /{version}/job/{job-uuid}/review`). A rejection **must** carry a comment — `400`
   otherwise, and `400` if the job is not `under review`. `get_job_reviews`
   (`GET /{version}/job/{job-uuid}/reviews`) returns the review history.
6. **Run it.** `create_job_run` (`POST /{version}/job/{job-uuid}/job-run`). Poll `get_job`
   (`GET /{version}/job/{job-uuid}`) and read `jobRuns[]`; each run has a `status` of
   `queued`, `running`, `succeeded` or `failed`, plus `runNumber`, `startedAt`, `finishedAt` and
   `isTestRun`.
7. **Collect output.** `get_job_run_results` (`GET /{version}/job-run/{job_run_uuid}/results`) —
   `400` until the job reaches **reencryption complete**. `get_job_run_logs`
   (`GET /{version}/job-run/{job_run_uuid}/logs`) — `400` unless the run is `succeeded` or
   `failed`; ask for `Accept: text/plain`.
8. **Abandon a run** with `cancel_job_run` (`POST /{version}/job-run/{job_run_uuid}/cancel`).
   `400` unless the run is `queued` or `running`.

## Rules

- **State machine, not free-form.** `draft -> under review -> accepted | rejected`; runs go
  `queued -> running -> succeeded | failed`. Most `400`s from this API are a state violation, not
  a payload problem — read `detail` before retrying.
- **`update_job` fails once the job is queued** and `delete_job` fails once a run has been
  submitted. Fix the job before you run it.
- **Retrieving results requires the `userIdentitySecret` cookie** as well as the bearer token.
- **There is no idempotency key.** `create_job_run` is not safe to blind-retry — poll `get_job`
  and inspect `jobRuns[]` before re-submitting.
- **There is no pagination.** `get_jobs_by_workspace` returns the whole list.
