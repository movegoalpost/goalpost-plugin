# ChatGPT plugin install paths

This is the ChatGPT plugin. OpenAI's plugin format is named after
Codex, the agent runtime inside ChatGPT and the `codex` CLI, so its
manifest, marketplace file, and commands all say `codex`; the same
plugin serves ChatGPT and the CLI. This directory ships two ways to
wire it up to Goalpost. The plugin path goes through a plugin
marketplace and is what most users should use. The manual
`config.toml` path is for CLI users who don't want to install a plugin
or are testing without one.

## Plugin path (recommended)

Install via the marketplace:

```bash
codex plugin marketplace add movegoalpost/goalpost-plugin
```

Then run `/plugins` in the Codex CLI and install **Goalpost**. (The
ChatGPT desktop app can add the same marketplace from
**Settings → Plugins → Add → Add a marketplace**. Until the plugin
is listed in OpenAI's plugin directory, most ChatGPT users are better
served by a custom MCP server; see the plugin README.) The
plugin bundles:

- The MCP server, the same native streamable-HTTP entry Claude Code
  uses (`../.mcp.json`, pointing at
  `https://api.movegoalpost.com/api/mcp`). Codex speaks to it
  directly; no local process sits in between.
- The Goalpost skills under `../skills/`, loaded into every Codex
  session that has the plugin enabled: `goalpost` (the mental model
  and conventions) and the `draft`, `review`, and `submit`
  workflows, which the model picks up when a request matches ("draft
  a change batch for…", "review my pending change batch", "submit the
  draft").

## What the Plugins tab shows

Everything on the plugin's page in the ChatGPT desktop app — the logo,
developer, website, privacy and terms links, capabilities, long
description — comes from the `interface` block in
`../.codex-plugin/plugin.json`. The logo and composer icon are files
inside the plugin (`../assets/`), regenerated from the brand SVG by
`npm run logo:rasters` in `marketing/`; the brand colors are the
per-theme accents from `docs/brand-color.md`.

Codex reads that block only once it has the plugin's files. A
marketplace entry whose `source` is a `url` is a "cross-repo" plugin:
before install the UI can show only what the marketplace entry itself
carries (name, category, description) and labels the rest as
unavailable. The Codex-native marketplace at
`../.agents/plugins/marketplace.json` therefore declares the plugin
with a `local` source (`./`), so the manifest is in the marketplace
checkout and the page is complete before install. The shared
`../.claude-plugin/marketplace.json` keeps the `url` source Claude Code
expects; Codex only falls back to it when no `.agents/plugins/`
marketplace exists.

## Signing in

Codex completes OAuth for a remote server on an explicit login rather
than on the first tool call. In the CLI:

```bash
codex mcp login goalpost
```

Codex identifies itself to Goalpost's OAuth server with the client
metadata document it publishes and opens the consent screen in your
browser; pick the workspace to bind. The
marketplace entry sets `policy.authentication: ON_INSTALL`, which asks
the ChatGPT desktop app to prompt for that sign-in when the plugin is
installed; if it does not, the CLI command above signs the same
installation in. Codex's own tracking of sign-in-on-first-call is
[openai/codex#23846](https://github.com/openai/codex/issues/23846).

## Manual `config.toml` path

Skip the plugin and add the snippet from
[`config.toml.example`](./config.toml.example) to
`~/.codex/config.toml`, then run `codex mcp login goalpost`. You get
the MCP server but not the skills.

## Other open questions

- **Does the desktop app prompt for sign-in at install?**
  `policy.authentication: ON_INSTALL` is documented for App and
  API-token auth; whether it also drives an MCP server's OAuth at
  install time is to be confirmed by hand. The CLI login covers the
  gap either way.
