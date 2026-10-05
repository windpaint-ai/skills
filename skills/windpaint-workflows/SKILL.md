---
name: windpaint-workflows
description: "Run Windpaint workflows, ready-made multi-step pipelines (for example text-to-clip) that take a small form and return named outputs. Use when the user wants a finished result that needs several generation steps, names a Windpaint workflow or preset, or asks for a clip from a text prompt. Load the windpaint skill first for auth and job basics."
---

# Run a workflow

A workflow runs server-side: it chains capabilities and local ops (resize, frame extraction,
audio mux, text overlay) and returns named outputs. Prefer a workflow over chaining `generate` calls by
hand when one fits the request.

## Steps

1. `list_workflows` (optionally `category="video"`, `"image"` or `"audio"`). For each workflow read:
   - `inputs`: the form, as `name -> {type, label, required, default, options}`. Types are `text`,
     `number`, `choice`, and `image`/`video`/`audio` (one asset id each).
   - `outputs`: the names you will get back (e.g. `video`, `still`, `cover`).
   - `available` and `unavailable_reasons`: a workflow whose steps have no served model cannot run.
     Tell the user which step blocks it; do not rebuild it by hand from unavailable capabilities.
   - `credits_estimate` at default settings.
2. Pick an available workflow that matches the outcome. At the time of writing `text-to-clip` is the
   only runnable one; others may be listed with `available: false`. Presets are workflows too: they
   pre-fill part of a base workflow's form.
3. Fill the form. Text and choice values are strings; a `choice` must be one of its `options`. Media
   inputs take one asset id, public https URL or `data:` URI (up to 5 MB) each; `upload_asset` first
   for a local file.
4. `estimate_workflow(slug, inputs)` and tell the user the credits and `available_credits` before
   running. Runs are refused with 402 if available credits are below the estimate.
5. `run_workflow(slug, inputs, wait=false)`, keep the run `id`, then `get_run(run_id, wait=true,
   timeout_s=600)`. Repeat `get_run` with `wait=true` while the status is `queued` or `running`.
   Runs with video take several minutes. Never start a second run because a wait timed out.
6. On `completed`, `outputs` maps each output name to assets. `download_asset` the ones the user wants
   (each returns a signed URL; save with `curl -fsSL -o <path> "<url>"`) and report paths, the
   workflow, and `credits.actual`.

## Progress and failures

- `get_run` lists each step's `status`, `request_id` and `error`. To show progress, report which steps
  are done. `list_recent_jobs(workflow_run_id=...)` lists the step jobs.
- If a step fails the run fails; `error` on the run and on that step says why. Credits already spent
  on completed steps are not returned. Local ops are free.
- Credits are checked when the run starts but held per step as each starts, so a run can still fail
  mid-way if the balance drops in the meantime.
- `cancel_run` cancels pending and running steps.
- From the CLI: `windpaint workflows`, `windpaint workflows get|estimate|run <slug> -i name=value`,
  and `windpaint runs` to list, inspect, wait for or cancel runs.

## Picking between a workflow and capabilities

- User wants the workflow's exact outputs (e.g. a clip plus its still and cover): run the workflow.
- User wants control over one step (a specific model, seed or resolution the form does not expose):
  compose capabilities with `windpaint-image` / `windpaint-video`.
