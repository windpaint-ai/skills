---
name: windpaint-api
description: "Write application code that calls the Windpaint REST API: submit generation jobs, poll or receive webhooks, upload inputs, fetch outputs, estimate credits, run workflows and handle errors. Use when the user is building Windpaint into an app, backend, script or pipeline (any language), debugging such an integration, or asks how a Windpaint endpoint works. Not for one-off generation from the agent; use the MCP tools for that."
---

# Integrate the Windpaint API

There is no published SDK yet. Write a thin client over HTTP in the app's language. The API reference
is at https://docs.windpaint.ai/api-reference/introduction; read it when you need a field this skill
does not cover, and do not invent endpoints.

## Basics

- Base URL: `https://api.windpaint.ai/v1`. Read the host from config (e.g. `WINDPAINT_API_URL`,
  default `https://api.windpaint.ai`), never hard-code it in several places.
- Auth: `Authorization: Bearer <key>`, where the key starts `aak_`. Keep the key server-side. Never
  ship it to a browser or mobile client; proxy through the app's backend.
- The version is the `/v1` path. There is no version header.
- Envelopes: submit, status and cancel of generation jobs return a bare JSON object. Every other
  endpoint wraps its payload as `{"data": ...}`.
- Errors: `{"error": {"code", "message", "details", "request_id"}}`. Branch on `code`, never on
  `message`. Log `request_id`; it maps to the server log line. Exception: 401/403 from the auth guard
  currently come back as `{"status": false, "error": "Unauthorized" | "Forbidden"}`, so branch on the
  HTTP status for those.

## Endpoints

```
GET  /generation/capabilities              capabilities -> slots -> models -> price per tier
GET  /generation/models                    models -> capabilities -> prices
POST /generation/estimate                  {capability, model?, prompt, inputs, resolution?, aspect_ratio?, quality?, duration?, project_id?}
POST /generation/uploads                   multipart, file field "data" -> asset
GET  /generation/assets[?limit]            GET /generation/assets/{id}
GET  /generation/assets/{id}/content       302 to a signed URL (about 15 min), or the bytes
POST /generation/capabilities/{capability} {model?, prompt, inputs: {slot: [asset_id]}, resolution?, aspect_ratio?,
                                            quality?, duration?, seed?, num_outputs?, output_format?,
                                            output_quality?, webhook_url?, webhook_events?, project_id?} -> 202
POST /generation/models/{model}            model-addressed alias: {prompt, image_urls, ...} -> 202
GET  /generation/requests[?limit&workflow_run_id]
GET  /generation/requests/{id}/status
POST /generation/requests/{id}/cancel
GET  /workflows[?category]                 GET /workflows/{slug}
POST /workflows/{slug}/estimate            {inputs, project_id?}
POST /workflows/{slug}/runs                {inputs, project_id?, webhook_url?, webhook_events?} -> 202
GET  /workflows/runs[?limit&workflow]      GET /workflows/runs/{id}    POST /workflows/runs/{id}/cancel
GET  /billing/balance                      available, held, by_source, next_expiry, lots
GET  /projects                             POST /projects {name, slug?, description?}
```

Prefer `/generation/capabilities/{capability}` over the model alias: its request shape is the same
for every model, so switching models is a one-field change.

## Submit and poll

```bash
curl -sS -X POST "$WINDPAINT_API_URL/v1/generation/capabilities/image.generate" \
  -H "Authorization: Bearer $WINDPAINT_API_KEY" -H "Content-Type: application/json" \
  -d '{"prompt": "a lighthouse at dusk, long exposure", "aspect_ratio": "16:9"}'
# 202 {"status": "queued", "request_id": "...", "status_url": "...", "cancel_url": "...", "credits_estimate": "..."}
```

Then `GET status_url` until `status` is terminal: `completed`, `failed`, `nsfw` or `canceled`
(`queued` and `in_progress` are not). Poll from about 2 s, backing off to about 10 s; video takes
minutes, images seconds. A completed status carries `outputs` (each `{id, url, content_type, width,
height, duration_s}`), the shortcuts `images` / `video`, and `credits: {estimate, actual}`. On
`failed`, `error` says why.

To skip polling for short jobs, send `Prefer: wait` (or `wait=N`, 1 to 60 s). If the job finishes in
time the response is 200 with the full status (check `status`: it can be `failed`, `nsfw` or
`canceled`); otherwise it is the usual 202 and you poll. `Preference-Applied: wait=N` says the API
waited. It works on workflow runs too (200 with `{"data": run}`), though runs with video take longer
than 60 s.

Image jobs take `num_outputs` (1 to the model's `max_outputs` from `/generation/capabilities`; video
models take only 1). The hold is the one-output price times `num_outputs`, and you are charged per
delivered output. `output_format` is `png` (default), `jpeg` or `webp`, with `output_quality` 1-100
for the last two; only models with non-empty `output_formats` accept it.

Credit amounts are decimals (serialized as strings); parse them as decimals, not floats.

## Webhooks

Pass `webhook_url` (absolute `https://`) on a job submit or a workflow run, and optionally
`webhook_events`: `completed` (default; any terminal status) and/or `started`. Windpaint POSTs
`{type, timestamp, data}`, where `data` is the job status or the run object and `type` is e.g.
`request.completed`, `request.failed`, `run.completed`.

- Deliveries are signed per [Standard Webhooks](https://www.standardwebhooks.com): headers
  `webhook-id`, `webhook-timestamp`, `webhook-signature`. Verify on the raw body with the
  organization's `whsec_` secret (`GET /webhooks/jobs/secret`); the `standardwebhooks` libraries do it.
  Reject timestamps more than 5 minutes off.
- Non-2xx answers (408, 429, 5xx, timeouts over 10 s) are retried up to 10 times over about 6 hours with
  the same `webhook-id`. De-duplicate on it. Other 4xx are not retried.
- Deliveries can arrive out of order; act on `data.status`. Answer 2xx fast and do slow work later.
- `GET /webhooks/jobs/deliveries[?request_id|run_id]` shows every attempt.

Keep a slow fallback poll for jobs still open long after they should have finished.

## Inputs and outputs

- Upload with multipart field `data`, free: PNG, JPEG or WebP images (20 MB, 64-8192 px), MP4, MOV or
  WebM video (200 MB, 60 s), MP3, WAV, M4A or OGG audio (25 MB, 5 min). The format is detected from
  the bytes. The response's `data.id` is the asset id. Limit failures are 422 `input.*`.
- Put asset ids in `inputs` by slot name from `/generation/capabilities`. Asset URLs this API returned
  are accepted too, and so are public https URLs (fetched at submit; failures are 422
  `input.fetch_failed`) and `data:` URIs up to 5 MB. Both are stored as new assets of the project.
- Assets belong to one project; an id from another project is a 422.
- Output `url`s point at `/v1/generation/assets/{id}/content`, which needs your key and redirects to a
  short-lived signed URL. Server-side, request it with the key, then follow the redirect **without**
  the `Authorization` header (or read `Location` with redirects disabled). Store the asset id, not the
  signed URL; mint a fresh one when you need it. Assets persist, so you do not have to copy them out.

## Projects

Every call runs in a project: `project_id` in the body, else the `X-Windpaint-Project` header (id or
slug), else the key's default project, else the organization's. A multi-tenant app can map each of
its own workspaces to a project and send the header per request.

## Cost

- `POST /generation/estimate` returns `{capability, model, credits, width, height, available}` without
  running anything. `credits: null` means the tier has no price and cannot be submitted. Use it to show
  prices in your UI and to gate expensive jobs.
- Submit holds the estimate and returns `credits_estimate`; the status's `credits.actual` is what was
  charged. `failed`, `nsfw` and `canceled` jobs are released in full.
- 402 `billing.insufficient_credits` has `details.required` and `details.available`. Surface it to the
  operator; do not retry.

## Errors and retries

| Status / code | Meaning | Retry? |
| --- | --- | --- |
| 401 | missing or bad key | no |
| 403 | key lacks the scope (generation read/write, billing read) | no |
| 404 `request.not_found` | wrong id, or another organization's | no |
| 409 `request.conflict` | e.g. writing to an archived project | no |
| 422 `request.validation_failed` | bad argument; `details` names it | no, fix the request |
| 402 `billing.insufficient_credits` | balance too low | no |
| 503 `request.service_unavailable` | model not available or tier unpriced here | no, pick another model/tier |
| 429, 500, 502, 504, network errors | transient | GETs, and POSTs sent with an `Idempotency-Key`: yes, with backoff and jitter (honor `Retry-After` on 429) |

Send an `Idempotency-Key` header (1-255 chars, e.g. a UUID per logical operation) on submits, uploads
and workflow runs. A retry with the same key and body within 24 hours returns the original response
with `Idempotent-Replayed: true` and creates and bills nothing new. Same key with a different body is
409 `idempotency.key_reused`; while the first is still processing, 409 `idempotency.in_progress` with
`Retry-After` (retry with the same key). Failed (4xx/5xx) responses are not stored. After a 5xx on a
submit, check `GET /generation/requests` before retrying, since a committed job can release the key.

## Checklist for a new integration

1. Key in server config, base URL configurable.
2. Read capabilities at startup or on a schedule; do not hard-code model names or prices.
3. Estimate before any user-triggered video job; show the cost.
4. Submit with an `Idempotency-Key`, store `request_id` with your own record, then poll, or receive verified webhooks.
5. Handle every terminal status, including `nsfw` and `canceled`.
6. Store asset ids; fetch content through your backend.
7. Map `error.code` to user-facing messages; log `error.request_id`.
