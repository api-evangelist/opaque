---
name: Onboard a dataset and share it into a workspace
description: Upload data into OPAQUE from local storage or a cloud bucket, describe its schema, and grant a workspace access to it.
api: openapi/opaque-platform-api-openapi.yml
operations: [upload_data, upload_data_aws_s3, upload_data_azure_blob_storage, upload_data_azure_files, upload_data_google_cloud_storage, upload_dataset_schema, list_data, get_data_metadata, update_data, share_org_data_with_workspace, get_data_by_workspace, get_datum, update_workspace_data, remove_workspace_data, download_data, delete_data, delete_datum]
generated: '2026-08-04'
method: generated
source: openapi/opaque-platform-api-openapi.yml
---

# Onboard a dataset and share it into a workspace

Data lands at the **organization** level, then is shared into **workspaces**. Encryption happens
client-side in the OPAQUE client/API pod before anything leaves the customer environment.

Authenticate first — see `opaque-authenticate.md`. Every operation here needs the
`userIdentitySecret` cookie in addition to the bearer token.

## Steps

1. **Upload.** Pick the operation that matches the source:
   - Local file: `upload_data` (`POST /{version}/organization/data/upload`), `multipart/form-data`.
   - `upload_data_aws_s3` (`POST /{version}/organization/data/upload_aws_s3`)
   - `upload_data_azure_blob_storage` (`.../upload_azure_blob_storage`)
   - `upload_data_azure_files` (`.../upload_azure_files`)
   - `upload_data_google_cloud_storage` (`.../upload_google_cloud_storage`)

   `400` on `upload_data` with `File already exists` means the name is taken. `500` covers the
   real failure modes and the `detail` field names them: could not write file to disk, size
   mismatch, encryption failed, upload to provider failed, or streaming from the source failed.
   Read `detail` — do not blind-retry, there is no idempotency key and a retry can duplicate.
2. **Describe the schema.** `upload_dataset_schema`
   (`POST /{version}/organization/data/schema`). `500` `Could not write schema to disk`.
3. **Confirm.** `list_data` (`GET /{version}/organization/datasets`) and `get_data_metadata`
   (`GET /{version}/organization/data`) return the dataset with `columns[]`, `dataSource`
   (`local | aws_s3 | azure_blob | azure_file | gcp | job_run_result`), `size`, `remoteUri`, and
   `testDataType` (`synthetic | mock | none`). Label test data honestly — `testDataType` is
   required.
4. **Share it.** `share_org_data_with_workspace`
   (`POST /{version}/organization/data/share`) shares a batch of organization datasets into a
   workspace. Verify with `get_data_by_workspace`
   (`GET /{version}/workspace/{workspace-uuid}/data`) and `get_datum`
   (`GET /{version}/datum/{datum-uuid}`).
5. **Maintain.** `update_data` (org level) and `update_workspace_data`
   (`PATCH /{version}/datum/{datum-uuid}`) amend metadata. `download_data`
   (`GET /{version}/organization/data/download`) retrieves it.
6. **Withdraw.** `remove_workspace_data`
   (`DELETE /{version}/workspace/{workspace-uuid}/data`) revokes a workspace's access without
   deleting the data. `delete_datum` and `delete_data` destroy it — `delete_datum` returns `500`
   `Provider deletion failed` if the underlying storage delete fails, which leaves the platform
   and the storage out of step. Verify after deleting.

## Rules

- Sharing is additive and revocable: revoking from a workspace is not a delete, and deleting at
  the organization level is not reversible.
- `remoteUri` is the exact path a query must use to reach the datum. Pass it through verbatim.
- A datum's access is governed by its `policy` (`DatumPolicy`) and the workspace's
  `workspacePolicy`. Do not assume a share grants query rights.
- No pagination — `list_data` returns everything.
