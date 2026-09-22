# Codex install paths

This plugin ships two ways to wire Codex up to Goalpost. The plugin
path goes through Codex's marketplace and is what most users should
use. The manual `config.toml` path is for users who don't want to
install a plugin or are testing without one.

## Plugin path (recommended)

Install via the marketplace:

```bash
codex plugin marketplace add movegoalpost/goalpost-plugin
```

Open the plugin directory in Codex, pick the marketplace, and enable
**Goalpost**. The plugin bundles:

- The MCP server, wired through `./codex/mcp.json` which uses an
  `npx mcp-remote` stdio shim to bridge to
  `https://api.movegoalpost.com/api/mcp` and complete the OAuth flow
  on first use.
- The Goalpost Skill at `../skills/goalpost/SKILL.md`, loaded into
  every Codex session that has the plugin enabled.

Slash commands aren't part of the Codex plugin spec yet, so Codex
users get the MCP server + skill but invoke the workflows by asking
in natural language ("draft a change batch for…", "review my pending
change batch", "submit the draft").

## Manual `config.toml` path

Skip the plugin and add the snippet from
[`config.toml.example`](./config.toml.example) to
`~/.codex/config.toml`. This is the same `npx mcp-remote` shim the
plugin uses internally; it works fine without going through the
marketplace.

## Why the `mcp-remote` shim instead of a native HTTP entry

Codex supports remote streamable-HTTP MCP servers (`url = "https://…"`
in `[mcp_servers.foo]`) but requires an explicit
`codex mcp login <name>` to complete OAuth — there's no automatic
RFC 9728 discovery on first tool call. Tracked in
[openai/codex#23846](https://github.com/openai/codex/issues/23846);
resolving that issue alone isn't enough to drop the shim. The shim
goes away when one of the following lands:

- Codex grows full RFC 9728 + Dynamic Client Registration support,
  matching what Claude Code already does.
- `policy.authentication: ON_INSTALL` on the marketplace entry is
  confirmed to drive `codex mcp login` for embedded MCP servers at
  install time. The Codex docs tie `ON_INSTALL` to App/API-token
  auth, not to MCP-server OAuth bootstrapping specifically, so this
  is unverified.

At that point `./codex/mcp.json` can be deleted and
`.codex-plugin/plugin.json`'s `mcpServers` can point at the root
`.mcp.json` — same fast path Claude Code uses, no subprocess.

## Other open questions

- **Will Codex's plugin spec grow slash commands?** Codex's
  documented manifest fields cover skills, hooks, apps, and MCP
  servers but not slash commands. If they're added, the existing
  `../commands/` files can be wired into `.codex-plugin/plugin.json`
  alongside the skill.
