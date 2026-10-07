# Windpaint skills

Agent skills for [Windpaint](https://windpaint.ai), the developer API for open-weight image and video
generation, asset storage and workflows. Packaged as a Claude Code plugin (and single-plugin marketplace), a Codex
plugin and a Cursor plugin. Installing the plugin also connects the Windpaint MCP server.

| Skill | Use it for |
| --- | --- |
| `windpaint` | Foundation: auth, capabilities and models, estimates, async jobs, assets, projects, balance |
| `windpaint-image` | Text to image (`image.generate`), image edit when a model is available |
| `windpaint-video` | Image to video (`video.generate` with a start frame) |
| `windpaint-workflows` | Ready-made multi-step workflows such as `text-to-clip` |
| `windpaint-api` | Writing app code against the REST API: submit, poll, webhooks, uploads, errors |

## Prerequisites

A Windpaint API key (`aak_...`) from the dashboard, exported in the shell that starts your agent:

```bash
export WINDPAINT_API_KEY=aak_...
```

The MCP config sends it as `Authorization: Bearer $WINDPAINT_API_KEY` to the hosted Windpaint MCP
server at `https://mcp.windpaint.ai/mcp`.

Website: https://windpaint.ai · Docs: https://docs.windpaint.ai/agents/overview

## Claude Code

```text
/plugin marketplace add windpaint-ai/skills
/plugin install windpaint@windpaint
```

Skills are then available as `/windpaint:windpaint`, `/windpaint:windpaint-image` and so on, and the
`windpaint` MCP server starts with the plugin. Run `/mcp` to check it connected. More:
https://docs.windpaint.ai/agents/claude-code

## Codex

```bash
codex plugin marketplace add windpaint-ai/skills
codex plugin add windpaint@windpaint
codex mcp get windpaint                               # should show bearer_token_env_var WINDPAINT_API_KEY
```

Codex reads the same `.claude-plugin/marketplace.json` and `.mcp.json`. It ignores the `headers` block
and takes the key from `bearer_token_env_var` instead, which is why `.mcp.json` carries both. More:
https://docs.windpaint.ai/agents/codex

## Cursor

Install the plugin from the Cursor marketplace once it is listed, or clone this repository and load it
as a local plugin. Set `WINDPAINT_API_KEY` in the plugin's configuration so the bundled MCP server can
authenticate. More: https://docs.windpaint.ai/agents/cursor

## Any agent (skills only)

```bash
npx skills add windpaint-ai/skills
```

This copies the `SKILL.md` files into the agent's skills directory. Add the MCP server separately, or
let the agent use the REST API as the `windpaint-api` skill describes. See
https://docs.windpaint.ai/agents/skills and https://docs.windpaint.ai/agents/mcp.

## Validate

```bash
claude plugin validate .
```
