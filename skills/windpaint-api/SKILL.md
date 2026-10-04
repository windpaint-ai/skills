---
name: windpaint-api
description: "Write application code that calls the Windpaint REST API: submit generation jobs, poll or receive webhooks, upload inputs, fetch outputs, estimate credits, run products and handle errors. Use when the user is building Windpaint into an app, backend, script or pipeline (any language), debugging such an integration, or asks how a Windpaint endpoint works. Not for one-off generation from the agent; use the MCP tools for that."
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
                                            quality?, duration?, seed?, webhook_url?, project_id?} -> 202
POST /generation/models/{model}            model-addressed alias: {prompt, image_urls, ...} -> 202
GET  /generation/requests[?limit&workflow_run_id]
GET  /generation/requests/{id}/status
POST /generation/requests/{id}/cancel
GET  /workflows/products[?category]        GET /workflows/products/{slug}
POST /workflows/products/{slug}/estimate   {inputs, project_id?}
POST /workflows/products/{slug}/runs       {inputs, project_id?} -> 202
GET  /workflows/runs[?limit&product]       GET /workflows/runs/{id}    POST /workflows/runs/{id}/cancel
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

Credit amounts are decimals (serialized as strings); parse them as decimals, not floats.

## Webhooks

Pass `webhook_url` on submit. When the job finishes, Windpaint POSTs the same body as the status
endpoint to that URL. Know its limits:

- It is **not signed**, is sent **once** with a 10 s timeout, and is **not retried**.
- In production only `https://` URLs are called.
- A canceled job may not send one. Product runs have no webhook.

So treat a webhook as a hint: take `request_id` from it, re-fetch `GET /generation/requests/{id}/status`
with your key before acting, and keep a poller or periodic reconcile for jobs that never report back.

## Inputs and outputs

- Upload with multipart field `data`: PNG, JPEG, WebP, MP4, MP3 or WAV, up to 50 MiB, free. The
  response's `data.id` is the asset id.
- Put asset ids in `inputs` by slot name from `/generation/capabilities`. Asset URLs this API returned
  are accepted too. Third-party URLs are not: download and upload them first.
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
| 429, 500, 502, 504, network errors | transient | GETs yes, with backoff and jitter |

Submits have **no idempotency key**. Blindly retrying a POST after a timeout can create and bill a second
job. Before resubmitting, list `GET /generation/requests?limit=...` and look for the job you meant to
create.

## Checklist for a new integration

1. Key in server config, base URL configurable.
2. Read capabilities at startup or on a schedule; do not hard-code model names or prices.
3. Estimate before any user-triggered video job; show the cost.
4. Submit, store `request_id` with your own record, then poll (and/or webhook + re-fetch).
5. Handle every terminal status, including `nsfw` and `canceled`.
6. Store asset ids; fetch content through your backend.
7. Map `error.code` to user-facing messages; log `error.request_id`.
