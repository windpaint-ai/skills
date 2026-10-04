---
name: windpaint-video
description: "Make a short video clip with Windpaint (capability video.generate), usually by animating a start-frame image. Use when the user asks Windpaint to animate an image or photo, make a clip, shot or motion version of a still, or turn a prompt into video. Estimates cost first and waits on the long-running job. Load the windpaint skill first for auth and job basics."
---

# Image to video

Video jobs take minutes and cost far more than images. Estimate first, submit once, then wait.

## Steps

1. `list_capabilities`, find `video.generate`. For each available model read `resolutions`,
   `durations`, the price per tier, and the model's own `inputs`: a `start_frame` with `min: 1` means
   the model needs a start image, and `end_frame` with `max: 0` means it takes no end frame.
2. Get the start frame:
   - The user gave a file or URL: `upload_asset` with `url`, or `data_base64` for a local file.
   - They gave an asset id or an earlier job's output: use that id.
   - They only gave a prompt: first check `list_products` for `text-to-clip` (see
     `windpaint-products`); it does still + clip in one run. Otherwise make a still with
     `windpaint-image` at the aspect ratio the video will use, and use its output id.
3. `estimate_cost(capability="video.generate", prompt=..., inputs={"start_frame": [id]},
   resolution=..., duration_s=..., aspect_ratio=...)`. Tell the user the credits and their available
   balance. Wait for a go-ahead unless they already approved spending on this task.
4. `generate(... same arguments ..., wait=false)`. Keep the returned `request_id`.
5. `wait_for_job(job_id=request_id, timeout_s=600)`. If it comes back still `queued`/`in_progress`,
   call `wait_for_job` again. Do not resubmit: the job is still running and still billed.
6. On `completed`, `download_asset` the `video` output (also in `outputs`) to get a signed URL, save it
   with `curl -fsSL -o <path> "<url>"`, and report the path, model, resolution, duration and credits.

## Prompting for motion

- The start frame fixes subject, look and composition. The prompt should describe **what moves and
  how the camera moves**, not re-describe the image: "slow push-in; steam rises from the cup; light
  flickers".
- One action per clip. Clips are short, so a single continuous motion reads best.
- Keep the start frame's aspect ratio equal to the requested `aspect_ratio` to avoid cropping.

## Failure handling

- `failed`: read `error`. Retry once only if it looks transient; credits for a failed job are released.
- `nsfw`: moderation blocked the output. Tell the user.
- 422 on submit: usually a missing required `start_frame`, an asset of the wrong kind in the slot, an
  asset from another project, or a duration/resolution the model does not list.
- To stop a job the user no longer wants: `cancel_job`.

## Not available today

Check `list_capabilities` rather than trusting this list, but at the time of writing: one image-to-video
model with a required start frame, a single 480p resolution and a single 5 s duration; no text-only
video model, no end-frame interpolation and no audio track on generated video.
