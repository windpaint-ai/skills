---
name: windpaint
description: "Foundation for using Windpaint, an image and video generation API. Covers auth, capabilities and models, picking a model from the live catalog, estimating credits before spending, async jobs and waiting, input assets and slots, projects, and the credit balance. Load this before any other windpaint-* skill, or when the user mentions Windpaint, WINDPAINT_API_KEY or Windpaint MCP tools."
---

# Windpaint

Windpaint runs generation **capabilities** (typed primitives such as `image.generate` or
`video.generate`) on **models** that implement them, and **workflows** (multi-step pipelines such as
`text-to-clip`) built from those capabilities. Everything is asynchronous and billed in credits.

## 1. Pick a surface

1. **MCP tools** (preferred when present): `list_capabilities`, `estimate_cost`, `upload_asset`,
   `generate`, `get_job`, `wait_for_job`, `download_asset`, `cancel_job`, `list_recent_jobs`,
   `list_assets`, `list_projects`, `create_project`, `get_balance`, `list_workflows`, `get_workflow`,
   `estimate_workflow`, `run_workflow`, `get_run`, `list_runs`, `cancel_run`.
2. **REST** with `curl` otherwise. Base URL `${WINDPAINT_API_URL:-https://api.windpaint.ai}/v1`,
   header `Authorization: Bearer $WINDPAINT_API_KEY`. Endpoints are listed in the `windpaint-api` skill.

Auth: an API key starting `aak_`, created in the Windpaint dashboard. Check it exists without
printing it: `[ -n "$WINDPAINT_API_KEY" ] && echo set || echo missing`. If missing, ask the user to
create a key and export it; never ask them to paste it into the chat.

## 2. Discover, never assume

The catalog changes. Do not rely on model names or prices from memory or from this file.

- Call `list_capabilities` (REST `GET /generation/capabilities`) first. For each capability it returns
  the input slots, whether a prompt is required, allowed aspect ratios, and the models with
  `available`, resolutions, durations, qualities and the credit price per tier.
- A capability with no `available` model cannot run. Say so; do not try another capability silently.
- Omitting `model` uses the capability's first available model. Pass `model` only when the user asks
  for one or a listed model fits better (resolution, duration, speed).
- Models with `licence_restricted: true` are evaluation only (see `licence_note`). Do not use them for
  work the user will ship or sell, and tell the user when one is the only option.
- Before building a chain of capabilities, check `list_workflows`: a workflow may already do it.

## 3. Estimate before spending

- Always estimate video jobs and workflow runs: `estimate_cost` / `estimate_workflow`
  (REST `POST /generation/estimate`, `POST /workflows/{slug}/estimate`). Estimates also
  validate the arguments and return the chosen model and output size.
- Report the credits and `available_credits` to the user before an expensive run, and before any
  batch. A null `credits` means that tier has no price and cannot be submitted.
- `available_credits` is missing when the key cannot read billing; call `get_balance` if you need it.
- A request the balance cannot cover fails with HTTP 402 `billing.insufficient_credits`
  (`details.required`, `details.available`). Top-ups happen in the dashboard; offer a cheaper option
  (lower resolution, shorter duration, another model) instead of retrying.
- Credits are held when a job starts and settled when it ends. `failed`, `nsfw` and `canceled` jobs
  release their hold in full.

## 4. Jobs are async

- Submitting returns `request_id` and `credits_estimate` at once (HTTP 202). Over REST, `Prefer: wait`
  (or `wait=N`, up to 60 s) holds the response until the job or run finishes and answers 200 with the
  final status; if it does not finish in time you get the usual 202 and poll.
- Job statuses: `queued`, `in_progress`, then one of `completed`, `failed`, `nsfw`, `canceled`.
  Workflow run statuses: `queued`, `running`, `completed`, `failed`, `canceled`.
- Image jobs take `num_outputs` (1 to the model's `max_outputs`, billed per delivered output) and
  `output_format` (`png`, `jpeg`, `webp`, with `output_quality` for the last two).
- Images take seconds. Video takes several minutes per clip. For video, submit with `wait=false`, then
  `wait_for_job` (max 600 s per call). If it returns still running, call `wait_for_job` again.
- **Never resubmit because a wait timed out.** The job is still running and still billed. Resubmit only
  after a terminal `failed`, and only once, after reading `error`.
- `nsfw` means the output was blocked by moderation. Tell the user; rephrase only if they agree.
- `cancel_job` stops billing for a job; a model call already running finishes but its output is
  discarded.

## 5. Assets and slots

- Inputs go in named slots: `inputs: {"<slot>": ["<asset_id>", ...]}`. Slot names and min/max counts
  come from `list_capabilities` (e.g. `start_frame` for `video.generate`, `images` for `image.edit`).
- Get asset ids from `upload_asset` or from an earlier job's `outputs[].id`. That is how steps chain.
- Public https URLs and `data:` URIs (up to 5 MB) also work as inputs; the API stores them as assets.
- Formats: PNG, JPEG, WebP images (20 MB); MP4, MOV, WebM video (200 MB, 60 s); MP3, WAV, M4A, OGG
  audio (25 MB, 5 min). Uploads are free.
- The hosted MCP server cannot read your disk: `upload_asset` takes `url` or `data_base64` (base64 of
  a local file).
- `download_asset` returns a signed URL valid about 15 minutes. To save the file, fetch it with
  `curl -fsSL -o <path> "<url>"` (no auth header needed on the signed URL). Assets persist in their
  project; there is no need to copy them elsewhere to keep them.

## 6. Projects

Every asset, job and run belongs to a project. Omitting `project` uses the API key's default project,
else the organization's default. `project` accepts an id or slug (REST: `project_id` in the body or the
`X-Windpaint-Project` header). For a distinct piece of work, `create_project` keeps its assets together.
Asset ids from one project cannot be used in another.

## 7. Reporting back

Tell the user what ran (capability, model, resolution/duration), the credits charged (`credits.actual`,
else the estimate), and where the output is (saved path or URL). Do not dump raw JSON.

## Live today

At the time of writing, only `image.generate` and `video.generate` (image to video, start frame
required) have available models, and `text-to-clip` is the only runnable workflow. Other capabilities
and workflows may be listed with `available: false`. Trust `list_capabilities` / `list_workflows` over
this note.

## Not available today

The hosted MCP takes an API key, not OAuth.

## Related skills

- `windpaint-image`: text to image.
- `windpaint-video`: animate an image into a clip.
- `windpaint-workflows`: run ready-made multi-step workflows.
- `windpaint-api`: write application code against the REST API.
