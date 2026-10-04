---
name: windpaint-image
description: "Generate images from a text prompt with Windpaint (capability image.generate), and edit or restyle an existing image when an image.edit model is available. Use when the user asks Windpaint for an image, still, frame, illustration, product shot, thumbnail or variations of one. Load the windpaint skill first for auth and job basics."
---

# Generate an image

## Steps

1. `list_capabilities` and find `image.generate`. Note its `aspect_ratios` and each available model's
   `resolutions`, `default_resolution` and price per resolution (at the time of writing, `1k` and
   `2k`). If the user wants to change an existing image, look at `image.edit` instead; it is only
   usable when one of its models has `available: true` (none does at the time of writing).
2. Pick settings from what is listed, not from memory:
   - `aspect_ratio`: from the capability's list (e.g. `16:9` for a video start frame, `9:16` for a
     vertical post).
   - `resolution`: the model's default unless the user needs larger. Higher tiers cost more.
   - `model`: omit to get the default; set it only for a reason you can state.
   - `seed`: set it when the user wants to reproduce or vary a result predictably.
3. For one image, skip the estimate unless the user asked about cost. For a batch, `estimate_cost`
   once, multiply by the count, and tell the user the total before submitting.
4. `generate(capability="image.generate", prompt=..., aspect_ratio=..., resolution=..., wait=true)`.
5. On `completed`, take `outputs[0].id` (also under `images`). `download_asset` it to get a signed
   URL, then save it with `curl -fsSL -o <path> "<url>"` to the path the user wants, or into the
   project's working directory.
6. Report the saved path, model, resolution and credits.

For `image.edit`: upload the source (and any references) with `upload_asset`, then
`generate(capability="image.edit", prompt="<instruction>", inputs={"images": [source_id, ...]})`.
Order in `images` matters: refer to them in the prompt in the same order. A `mask` slot (one mask
asset) limits the change to a region.

## Prompting

- Describe subject, setting, composition, lighting and style in plain sentences. Put the subject first.
- Say what text should appear verbatim in quotes if the image needs text; keep it short.
- For a video start frame, compose for motion: leave space in the direction the subject will move,
  and match the aspect ratio the video will use.
- To iterate, change one thing per attempt and keep the seed.

## Variations

Submit several jobs with different seeds (or prompt tweaks) with `wait=false`, then `wait_for_job` each.
Estimate the batch first. Do not submit more than the user asked for.

## Errors

- 422 `request.validation_failed`: an argument is outside what the model lists (resolution, aspect
  ratio). Re-read `list_capabilities` and fix the argument.
- 503 `request.service_unavailable`: no model or price for that request here. Try the capability's
  other listed models or report it.
- 402: see the windpaint skill, section 3.
