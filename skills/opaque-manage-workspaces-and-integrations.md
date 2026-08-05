---
name: Manage workspaces, members and reusable integrations
description: Create a workspace, manage its membership and organization roles, and share reusable data/LLM connector configurations into it.
api: openapi/opaque-platform-api-openapi.yml
operations: [create_workspace, get_workspaces, get_workspace, update_workspace, respond_workspace_invite, get_shared_workspace_members, invite_organization_member, get_organization_invitations, get_organization_invitation, get_organization_members, update_user_organization_roles, create_asset_config, get_asset_configs_by_organization, share_asset_configs_with_workspaces, get_asset_configs_by_workspace, get_asset_config, edit_asset_config, revoke_asset_config_from_workspace, delete_asset_config]
generated: '2026-08-04'
method: generated
source: openapi/opaque-platform-api-openapi.yml
---

# Manage workspaces, members and reusable integrations

A workspace (a "consortium" in the underlying model) is the collaboration boundary — it can span
organizations. Asset configs are reusable connectors (a data source or an LLM) that an admin
defines once and shares into workspaces.

Authenticate first — see `opaque-authenticate.md`.

## Workspaces

1. `create_workspace` (`POST /{version}/workspace`). `400` means a member does not exist or the
   name does not match the required format.
2. `get_workspaces` / `get_workspace` read them; `update_workspace`
   (`PATCH /{version}/workspace/{workspace-uuid}`) amends. `status` runs
   `pending -> verified | rejected | failed`; an `archived` flag retires a workspace rather than
   deleting it.
3. `version` on a workspace is the **lowest** OPAQUE version across all members. In a multiparty
   workspace only features common to every member's version are available. Check
   `suggestedComputeVersion` before assuming a capability is present.
4. `get_shared_workspace_members` (`GET /{version}/workspaces/members`) lists who you share with.

> **Version note.** `respond_workspace_invite`
> (`POST /{version}/workspace/{workspace-uuid}/invite`) is declared against API version 2.5, but
> OPAQUE 2.7.0 removed the workspace invite flow in favour of fully mutable membership. Confirm
> your deployment version before building against it.

## Organization membership

- `invite_organization_member` (`POST /{version}/organization/invite`) invites by email;
  `get_organization_invitations` and `get_organization_invitation` track them;
  `get_organization_members` lists the org.
- `update_user_organization_roles` (`PATCH /{version}/user/organization-roles`) changes another
  user's roles. `400` — the organization admin role can neither be added nor removed through the
  API.

## Asset configs (integrations)

1. `create_asset_config` (`POST /{version}/asset-config`). `assetType` is `data_connector` or
   `llm_connector`; `definition` is a **base64-encoded serialized protobuf**.
2. `get_asset_configs_by_organization` (`GET /{version}/organization/asset-configs`) and
   `get_asset_config` (`GET /{version}/asset-config/{asset_config_uuid}`, `404` if absent) read
   them; `edit_asset_config` (`PATCH`) amends.
3. `share_asset_configs_with_workspaces` (`POST /{version}/organization/asset-configs/share`)
   shares a batch; `get_asset_configs_by_workspace`
   (`GET /{version}/asset-configs/workspace/{workspace_uuid}`) confirms.
4. `revoke_asset_config_from_workspace`
   (`DELETE /{version}/workspace/{workspace-uuid}/asset-config/{asset-config-uuid}`) withdraws
   access; `delete_asset_config` destroys it — `403` if the caller lacks permission, `404` if it
   is not there.

## Rules

- Prefer revoking to deleting. An asset config in use by a live workflow should be revoked from
  the workspace, not deleted from the organization.
- Name integrations for the people who will reuse them ("Customer Search Index", not
  "Azure Test 1") — OPAQUE's own guidance, and it is the difference between a reused connector
  and a duplicated one.
- Roles and workspace policy, not token scopes, decide what a caller may do. A `403` is an
  authorization decision, not a bad token.
