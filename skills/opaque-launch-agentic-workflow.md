---
name: Build, approve and launch an agentic workflow
description: Take an OPAQUE agentic workflow from draft through review to live, then invoke it and shut it down.
api: openapi/opaque-platform-api-openapi.yml
operations: [get_workspaces, create_workflow, get_workflow, update_workflow, request_workflow_review, cancel_workflow_review, review_workflow, get_workflow_reviews, launch_workflow, shutdown_workflow, get_workspace_workflows, delete_workflow]
generated: '2026-08-04'
method: generated
source: openapi/opaque-platform-api-openapi.yml
---

# Build, approve and launch an agentic workflow

An OPAQUE workflow is a graph of nodes — LLM services, retrievers, agents, and utilities such as
the MCP Tool and the Redact/Unredact services — that runs inside a trusted execution environment.
Like jobs, workflows are approval-gated.

Authenticate first — see `opaque-authenticate.md`.

## Steps

1. **Choose an agentic workspace.** `get_workspaces` (`GET /{version}/workspaces`) and pick one
   whose `type` is `agentic`. `get_workspace_workflows`
   (`GET /{version}/workspace/{workspace-uuid}/workflows`) lists what is already there.
2. **Create the workflow.** `create_workflow`
   (`POST /{version}/workspace/{workspace-uuid}/workflow`). It starts in `draft`.
3. **Shape the graph.** `update_workflow` (`PATCH /{version}/workflow/{workflow-uuid}`).
   `400` means an invalid workflow status or an invalid body parameter. Read it back with
   `get_workflow` (`GET /{version}/workflow/{workflow-uuid}`).
4. **Request approval.** `request_workflow_review`
   (`POST /{version}/workflow/{workflow-uuid}/request-review`). `400` means the workflow graph is
   invalid — fix the graph, not the request. Withdraw with `cancel_workflow_review`
   (`POST /{version}/workflow/{workflow-uuid}/cancel-review`).
5. **Approve or reject.** A reviewer calls `review_workflow`
   (`POST /{version}/workflow/{workflow-uuid}/review`). `400` unless the workflow is
   **under review**. `get_workflow_reviews` (`GET /{version}/workflow/{workflow-uuid}/reviews`)
   returns the trail.
6. **Launch.** `launch_workflow` (`POST /{version}/workflow/{workflow-uuid}/launch-workflow`).
   Status moves `accepted -> launching -> live`. If Test mode was enabled in draft, it launches
   into `Testing` instead — see `sandbox/opaque-sandbox.yml`.
7. **Invoke it.** Invocation is **not** part of this REST API. Use the OPAQUE Python SDK against
   the workflow service:
   `WorkflowService(workflow_uuid=uuid.UUID(...)).submit({...})`, with `OPAQUE_DATAPLANE_DOMAIN`,
   `OPAQUE_REST_URL` and `OPAQUE_API_KEY` set. The SDK talks to the workflow over attested TLS and
   will refuse to submit if attestation fails. Pass `request_report=True` to receive and appraise
   an attestation report; `report_path` and `appraisal_path` persist the evidence.
8. **Shut down.** `shutdown_workflow` (`POST /{version}/workflow/{workflow-uuid}/stop-workflow`).
   `delete_workflow` (`DELETE /{version}/workflow/{workflow-uuid}`) only works in `draft` — `400`
   otherwise.

## Rules

- Workflow states: `draft -> under review -> accepted | rejected -> launching -> live ->
  shutting down | failed`. Every `400` in this flow is a state or graph violation.
- The input payload of `submit()` must match the schema expected by the Start node and its
  connected nodes. There is no published schema for it in the REST contract — read it off the
  workflow graph.
- Attestation reports are produced when an aTLS connection is established, not per request.
  Connections are pooled, so many invocations produce no new report. Do not treat a missing
  report as a failed attestation.
- Test mode traces can expose sensitive data. Do not leave a production workflow in Test mode.
