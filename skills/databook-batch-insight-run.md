---
name: Run a Databook batch insight job
description: >-
  Pick insights from Databook's catalogue, upload a CSV of target accounts as a batch job,
  poll the job to completion, and fetch the result file. Use this instead of chat or reasoning
  whenever the work covers more than a handful of accounts, because any single Databook API
  call is cut off at 180 seconds.
api: openapi/databook-openapi-original.json
base_url: https://api.databook.com
operations:
  - list_insights_v1_insights_get
  - get_job_input_csv_template_v1_batch_job_input_csv_template_get
  - create_job_v1_batch_job_post
  - get_job_by_id_v1_batch_job__job_id__get
  - get_job_result_by_id_v1_batch_job__job_id__result_get
generated: '2026-08-13'
method: generated
source: openapi/databook-openapi-original.json
---

# Run a Databook batch insight job

Databook answers a fixed catalogue of analytical questions ("insights") about target companies.
For anything beyond a single ad-hoc question, the batch job endpoints are the correct surface:
they run asynchronously and are not subject to the 180-second per-call ceiling that applies to
`/v1/chat` and `/v1/reasoning`.

## Before you start

- **Credentials are issued by a human.** There is no self-serve key and no OAuth flow. The
  bearer token is provisioned by Databook support. If you do not have one, stop — you cannot
  obtain one programmatically.
- Every request needs all three headers:
  - `Authorization: Bearer <YOUR_ACCESS_TOKEN>`
  - `databook-user-id: <user id>`
  - `databook-tenant-id: <tenant id>`
  The reference states the two identity headers are required on every call even though the
  OpenAPI declares them optional. Always send them.

## Steps

1. **Choose the insights.** Call `list_insights_v1_insights_get`
   (`GET /v1/insights`). Each element in `result` is `{id, name, version, description,
   category}`. The `id` is a UUID and it is what the batch input file references. Categories in
   the published catalogue are Competitors, Firmographics, Account Planning, Meeting Preparation
   and Target Account. Select by `category` and `description`, and keep the `version` you saw —
   insights are versioned, so a rerun months later may not be the same question.

2. **Get the input format.** Call
   `get_job_input_csv_template_v1_batch_job_input_csv_template_get`
   (`GET /v1/batch/job/input-csv-template`). The response is
   `{job_input_csv_template_url}` — download that URL and use it as the schema for your upload.
   Do not hand-roll the CSV columns; the template is the contract, and it is the only place the
   column set is published.

3. **Submit the job.** Call `create_job_v1_batch_job_post` (`POST /v1/batch/job`) with
   `multipart/form-data` and a single `file` part containing your populated CSV. This is the one
   endpoint that is not JSON-encoded; do not set `Content-Type: application/json` on it. The
   response is the job object: `{id, state, stopped_reason, created_at, started_running_at,
   stopped_at}`. Persist `id`.

   **There is no idempotency key.** Databook publishes no `Idempotency-Key` header or any other
   deduplication contract, so a retried submission creates a second job and consumes the work
   twice. If the call fails without a response body, call `list_job_v1_batch_job_get` and match
   on `created_at` before resubmitting.

4. **Poll to completion.** Call `get_job_by_id_v1_batch_job__job_id__get`
   (`GET /v1/batch/job/{job_id}`) and read `state`. There are no webhooks and no callbacks —
   polling is the only completion signal Databook offers. Back off between polls: the API is
   rate limited, no limit number is published, and exceeding it returns
   `429 rate_limit_error`. When the job stops, `stopped_reason` and `stopped_at` explain how.

5. **Fetch the result.** Call `get_job_result_by_id_v1_batch_job__job_id__result_get`
   (`GET /v1/batch/job/{job_id}/result`). The response is `{job_id, job_result_s3_url}` — the
   output is a file at an S3 URL, not an inline payload. Download it promptly; treat the URL as
   short-lived and never log or persist it, as it is a direct object link.

## Error handling

Every operation returns the same envelope:

```json
{ "error": { "type": "invalid_request", "message": "Invalid request" }, "reference_id": "..." }
```

| Status | `error.type` | What to do |
| --- | --- | --- |
| 400 | `invalid_request` | Fix the payload against the schema. Do not retry unchanged. |
| 401 | `invalid_authentication_error` | Token missing, invalid, or lacking permission — a human must fix it. Do not retry. |
| 404 | `resource_not_found_error` | Wrong `job_id`, or it belongs to another tenant. |
| 429 | `rate_limit_error` | Exponential backoff. No `Retry-After` is documented, so choose your own base delay. |
| 500 | `internal_server_error` | Retry; keep `reference_id` for Databook support. |
| 503 | `server_side_overload_error` | Overload, or the call exceeded 180s. Retry later. |

## Do not

- Do not loop `/v1/chat` or `/v1/reasoning` across an account list to emulate a batch — that is
  what the 180-second ceiling and the batch endpoints exist to prevent.
- Do not invent `insight_id` values. Only ids returned by `list_insights_v1_insights_get` (or
  listed in the spec's `insights-catalog` tag) are valid.
- Do not expect paging. `GET /v1/batch/job` and `GET /v1/insights` return unbounded `result`
  arrays with no limit, offset or cursor parameter.
