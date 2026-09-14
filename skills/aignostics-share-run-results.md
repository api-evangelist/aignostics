---
name: aignostics-share-run-results
description: >-
  Share an Aignostics run with another user, organization, or an external collaborator via a
  revocable, expiring share token - and revoke it cleanly.
api: Aignostics Platform API
base_url: https://platform.aignostics.com/api/v1
operations:
  - create_grant_v1_access_grants_post
  - list_grants_v1_access_grants_get
  - revoke_grant_v1_access_grants__grant_id__delete
  - create_share_token_v1_access_share_tokens_post
  - list_share_tokens_v1_access_share_tokens_get
  - revoke_share_token_v1_access_share_tokens__share_token_id__delete
generated: '2026-09-14'
method: generated
source: openapi/aignostics-platform-api-openapi.json
---

# Share a run

Aignostics models access as relation tuples rather than OAuth scopes. A **grant** binds a
`resource_type` (`run`, `item`, `output_artifact`, `share_token`) and `resource_id` to a
`subject_type` (`user`, `organization_admin`, `organization_user`, `share_token`) with a `relation`
(`owner`, `editor`, `viewer`).

**Only `viewer` grants can be created through the API.** Anything else returns `422 Unprocessable
Entity - Only viewer grants can be created`.

## Share with someone inside the platform

`create_grant_v1_access_grants_post` (`POST /v1/access/grants`) with `resource_type: run`, the
`resource_id`, a `subject_type` and either `subject_id` or `subject_email`, and
`relation: viewer`.

`403` means you do not own the resource. `404` means the resource does not exist or is not visible
to you — the API deliberately does not distinguish those.

Audit with `list_grants_v1_access_grants_get` (`GET /v1/access/grants`), filtering on
`resource_id`, `subject_id`, `relation` or `revoked`. Organization admins see every grant in the
organization; everyone else sees grants on resources they submitted.

## Share with someone outside it

1. `create_share_token_v1_access_share_tokens_post` (`POST /v1/access/share-tokens`) with an
   `expires_at`. Always set one — it is caller-supplied and there is no documented default.
2. The response contains `share_token`. **The value is shown only once and is never stored.** Hand it
   to its recipient in the same operation that created it, or you will have to revoke and reissue.
3. `create_grant_v1_access_grants_post` with `subject_type: share_token`, `subject_id` set to the
   `share_token_id`, `relation: viewer`, and the run as the resource.

The holder then passes `?share_token=...` on `get_run_v1_runs__run_id__get`,
`list_run_items_v1_runs__run_id__items_get`, `get_item_by_run_...` and
`get_artifact_url_...` — the four read operations that accept it.

## Revoke

- `revoke_grant_v1_access_grants__grant_id__delete` (`DELETE /v1/access/grants/{grant_id}`) sets
  `revoked_at`. `409` means it was already revoked — that is success, not failure.
- `revoke_share_token_v1_access_share_tokens__share_token_id__delete`
  (`DELETE /v1/access/share-tokens/{share_token_id}`) "invalidates the credential regardless of any
  active grants". Use this one when a token may have leaked: it kills the credential itself rather
  than one path to it.

Both are reversible in the sense that matters — you can always re-share — and neither destroys data.
A grant has no expiry of its own, so it persists until revoked; a share token expires at
`expires_at` on top of being revocable.
