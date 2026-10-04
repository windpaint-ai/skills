# Windpaint skills: agent notes

## Using Windpaint from this repo

If you are an agent (Codex, Cursor, or any tool that reads `AGENTS.md`) and the user wants to generate
media with Windpaint or build it into an app:

1. Read `skills/windpaint/SKILL.md` first, then the skill for the task:
   - `skills/windpaint-image/SKILL.md`: text to image (and image edit when available).
   - `skills/windpaint-video/SKILL.md`: animate an image into a clip.
   - `skills/windpaint-products/SKILL.md`: ready-made multi-step products.
   - `skills/windpaint-api/SKILL.md`: application code against the REST API.
2. Check `WINDPAINT_API_KEY` is set without printing it. If not, ask the user to create a key in the
   Windpaint dashboard and export it.
3. If the Windpaint MCP server is not connected, suggest adding it (see `README.md`), or use the REST
   API as described in `skills/windpaint-api/SKILL.md`.
4. Discover models and prices live (`list_capabilities`); estimate before video jobs and product runs.

## Files

- `skills/*/SKILL.md`: the skills. Frontmatter has `name` (same as the directory) and `description`.
- `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json`: Claude Code plugin and marketplace.
- `.codex-plugin/plugin.json`: Codex plugin manifest.
- `.cursor-plugin/plugin.json`: Cursor plugin manifest.
- `.mcp.json`: the hosted Windpaint MCP server, authenticated with `WINDPAINT_API_KEY`.

## Editing guidelines

- Only document what the Windpaint API and MCP server do today. Planned features stay out until they
  ship.
- Do not hard-code model names, prices or tier lists as facts; tell the agent to read them from
  `list_capabilities` / `list_products`.
- Keep skills short and agent-facing. No marketing copy.
- Keep the version in the three plugin manifests and `marketplace.json` in step.
- Validate with `claude plugin validate .`.

## Mirroring

This repository is a read-only mirror, published from Windpaint's internal source tree, where these
files are maintained. Changes made directly here are overwritten on the next sync. Maintainers edit
the source copy, validate it, and run `just publish-skills` there, which replaces this repository's
contents with the source copy as a single new commit on `main`.
