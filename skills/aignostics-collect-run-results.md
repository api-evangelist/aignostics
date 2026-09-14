---
name: aignostics-collect-run-results
description: >-
  Retrieve per-slide output artifacts from a terminated Aignostics run through short-lived signed
  storage URLs, inside the 30-day retention window.
api: Aignostics Platform API
base_url: https://platform.aignostics.com/api/v1
operations:
  - list_run_items_v1_runs__run_id__items_get
  - get_item_by_run_v1_runs__run_id__items__external_id__get
  - get_artifact_url_v1_runs__run_id__artifacts__artifact_id__file_get
  - delete_run_items_v1_runs__run_id__artifacts_delete
generated: '2026-09-14'
method: generated
source: openapi/aignostics-platform-api-openapi.json
---

# Collect results from a run

## 1. Enumerate the items

`list_run_items_v1_runs__run_id__items_get` (`GET /v1/runs/{run_id}/items`) is paginated: `page`
(default 1) and `page_size` (default 50, min 5, max 100), with a `sort` parameter taking `+field` or
`-field`. Filter with `state`, `termination_reason`, `external_id__in`, `item_id__in`, or a
PostgreSQL JSONPath expression over `custom_metadata` — for example `$.priority ? (@ == "high")`.

Each item carries `output_artifacts[]`, and each of those has an `output_artifact_id`, a
`mime_type` (`application/vnd.apache.parquet`, `application/json` or `image/tiff`), a `state` and a
`termination_reason`.

Check `termination_reason` per item before trusting the output. `SUCCEEDED` is the only value that
means a usable result; `USER_ERROR`, `SYSTEM_ERROR` and `SKIPPED` each leave `error_code` and
`error_message` populated. A run can be `TERMINATED` with `output: PARTIAL` — some slides succeeded
and some did not.

To fetch one slide by your own identifier instead, use
`get_item_by_run_v1_runs__run_id__items__external_id__get`
(`GET /v1/runs/{run_id}/items/{external_id}`).

## 2. Download each artifact

`get_artifact_url_v1_runs__run_id__artifacts__artifact_id__file_get`
(`GET /v1/runs/{run_id}/artifacts/{artifact_id}/file`) responds with a **307 redirect to a presigned
Google Cloud Storage URL**.

Two rules follow from that:

- Your HTTP client must follow redirects, and must not send the `Authorization` header to the
  redirect target.
- The signed URL "is valid for a limited time, so it should be used immediately". Never store it,
  never queue it for later, never hand it to another system that will use it minutes from now. Ask
  again instead — the operation is a read and is safe to repeat.

A `410 Gone` means the artifact has been deleted and cannot be recovered. The run would have to be
resubmitted, and rebilled.

## 3. Respect the retention window

**Artifacts are deleted automatically 30 days after the run finishes, regardless of whether anyone
requests deletion.** Treat run completion as the start of a 30-day clock and copy anything you need
into your own storage inside it.

## 4. Deleting early is one-way

`delete_run_items_v1_runs__run_id__artifacts_delete` (`DELETE /v1/runs/{run_id}/artifacts`) removes
the outputs permanently. It is callable only once the run is `TERMINATED` — calling it earlier
returns `409`. A second call returns `410`. There is no undo and no restore window.

Only call it when you have confirmed the artifacts are safely copied. If you are automating cleanup,
verify your own copy first; the 30-day auto-delete will do the job anyway if you do nothing.
