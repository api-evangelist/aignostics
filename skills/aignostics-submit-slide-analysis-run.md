---
name: aignostics-submit-slide-analysis-run
description: >-
  Submit whole slide images to an Aignostics computational pathology application, poll until the run
  terminates, and collect per-slide results - without double-billing a retry.
api: Aignostics Platform API
base_url: https://platform.aignostics.com/api/v1
operations:
  - list_applications_v1_applications_get
  - application_version_details_v1_applications__application_id__versions__version__get
  - create_run_v1_runs_post
  - get_run_v1_runs__run_id__get
  - list_run_items_v1_runs__run_id__items_get
  - cancel_run_v1_runs__run_id__cancel_post
generated: '2026-09-14'
method: generated
source: openapi/aignostics-platform-api-openapi.json, https://aignostics.readthedocs.io/en/latest/get_started_api.html
---

# Submit a slide analysis run

## Before you start

You need an access token. There is no anonymous access and no API key. Get one with the device-code
flow against `https://aignostics-platform.eu.auth0.com` using the `client_id` your organization
received from `support@aignostics.com`, then send `Authorization: Bearer {access_token}` on every
call.

## 1. Find the application and pin its version

Call `list_applications_v1_applications_get` (`GET /v1/applications`). Each entry carries
`application_id`, `regulatory_classes` and the versions available to you.

**Read `regulatory_classes` before you use any result.** The values are `RUO`, `IVDR` and `FDA`. An
application marked `RUO` is Research Use Only and its output must not be used clinically.

Then call `application_version_details_v1_applications__application_id__versions__version__get`
(`GET /v1/applications/{application_id}/versions/{version}`). This response is the real contract for
the work you are about to submit: `input_artifacts[]` tells you the accepted MIME types
(`application/dicom`, `application/zip`, `application/octet-stream`, `image/tiff`) and gives a
`metadata_schema` — a JSON Schema your per-slide metadata must validate against. Validate locally
against that schema; there is no dry-run endpoint, so an invalid payload only surfaces as a 400 or
422 after you submit.

Always pass an explicit `version_number` on the run. Omitting it selects "the latest version
available to you", which makes the run non-reproducible the moment Aignostics releases a new version.

## 2. Submit the run — the one call that spends money

Call `create_run_v1_runs_post` (`POST /v1/runs`) with `application_id`, `version_number`, and
`items[]`, one item per whole slide image. Each item needs:

- `external_id` — your own identifier. It must be unique within the run and it is the only join key
  back to your case or specimen record. A duplicate returns 400.
- `input_artifacts[]` — each with a `download_url` the platform can reach, plus `metadata` matching
  the version's `metadata_schema`.
- `custom_metadata` — optional JSON you can later filter on with JSONPath.

Optional but worth setting:

- `scheduling.deadline` — a hard deadline. "The run will be cancelled if not completed by this time."
  This is a real spend guardrail; use it on any unattended submission.
- `callback_context` — opaque correlation JSON, max 1024 bytes.

**This operation is not idempotent.** The documentation is explicit: "POST /v1/runs is not
idempotent — calling it twice analyzes your slides twice." There is no `Idempotency-Key` header in
this API. So:

1. Persist your intent (application, version, the list of `external_id`s) **before** you send.
2. Send once. Persist the returned `run_id` from the 201 **immediately**.
3. If the call fails without a response body, do **not** blindly resubmit. Call
   `list_runs_v1_runs_get` filtered by `external_id` (and by `custom_metadata` if you set a
   correlation key) to find out whether the run actually landed. Only submit again if it did not.

A `402 Payment Required` means a quota ceiling — slides per run, or monthly slides — would be
exceeded. The numbers are not published. Reduce the batch or stop; do not retry.

Never retry a 4xx. Retry 5xx, timeouts and connection errors with exponential backoff and jitter.

## 3. Poll until terminal

Call `get_run_v1_runs__run_id__get` (`GET /v1/runs/{run_id}`). `state` is `PENDING`, `PROCESSING` or
`TERMINATED`; `statistics` breaks items down into pending, processing, succeeded, skipped,
user-error and system-error counts. `output` tells you whether results are `NONE`, `PARTIAL` or
`FULL`.

Do not wait for the whole run before collecting. Items terminate independently — poll
`list_run_items_v1_runs__run_id__items_get` (`GET /v1/runs/{run_id}/items`) and collect each item as
it reaches `TERMINATED` with `termination_reason: SUCCEEDED`. `queue_position_org` and
`queue_position_platform` tell you how deep the queue is, so back your polling interval off when they
are large.

There is no webhook and no AsyncAPI channel to subscribe to. Polling is the only mechanism.

## 4. Abort if you need to

`cancel_run_v1_runs__run_id__cancel_post` (`POST /v1/runs/{run_id}/cancel`) works "any time while
the run is not in the terminated state". Pending items are not processed and do not add to the cost;
already-completed items stay downloadable. A `409` means it is already cancelled — treat that as
success, not as an error.

## 5. Collect results before the window closes

See `aignostics-collect-run-results`. The deadline that matters: **artifacts are deleted
automatically 30 days after the run finishes**, whether or not anyone asks.
