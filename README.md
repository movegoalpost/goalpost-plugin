# Goalpost plugin for AI assistants

> **Status: pre-release.** This directory is the source of truth that
> will mirror to [`movegoalpost/goalpost-plugin`](https://github.com/movegoalpost/goalpost-plugin).
> Until that public repo exists, the marketplace install paths below
> won't resolve — use the `--plugin-dir` form against a local checkout
> to test.

Connect [Goalpost](https://www.movegoalpost.com) — versioned
specification documents — to your AI assistant. Once installed, the
assistant can read your projects, draft and revise change batches, and
manage project-level team access on your behalf.

Requires a **Pro-tier Goalpost workspace**. Free-tier workspaces
don't appear in the consent screen's workspace picker, so a user with
no Pro workspace can't complete the connection.

## What you (or your assistant) can do

- List projects, read the latest approved specification, and browse
  change batch history.
- Create a new project.
- Create draft change batches, iterate on them, and submit them into the
  approval queue.
- List teams in the workspace, see which teams are assigned to a
  project, and assign/revoke that access.

## What this connection cannot do

By design, the following actions are not exposed and must be done in
the Goalpost webapp:

- Cast approval votes (`approve`, `reject`, `withdraw`). The plugin
  *can* submit a draft for approval, but it can't cast votes on it.
- Configure approval rules.
- Delete projects or change batches.
- Create, rename, delete teams, or change team membership.
- Anything under workspace settings or billing.
- Authentication or token management.

## Install

### Claude Code

Once the public repo is published, install via the marketplace:

```bash
claude plugin marketplace add movegoalpost/goalpost-plugin
claude plugin install goalpost@movegoalpost
```

Or from inside a Claude Code session:

```
/plugin marketplace add movegoalpost/goalpost-plugin
/plugin install goalpost@movegoalpost
```

To test against a local checkout (no marketplace needed):

```bash
claude --plugin-dir /path/to/goalpost-plugin
```

On first use of any `goalpost:*` tool, Claude Code will pop the
Goalpost OAuth consent screen in your browser. Pick the workspace you
want this connection bound to and approve — the assistant can now
call Goalpost tools for that workspace.

### Codex CLI / desktop

Once the public repo is published, install via the marketplace:

```bash
codex plugin marketplace add movegoalpost/goalpost-plugin
```

Open the plugin directory in Codex, pick the marketplace, and enable
**Goalpost**.

Codex bundles the MCP server through an `npx mcp-remote` stdio shim
that bridges to Goalpost's HTTPS endpoint and handles OAuth on first
use locally; we'll switch to a native streamable-HTTP entry once
Codex's MCP client confirms support. The Goalpost Skill rides along
with the plugin; slash commands (`/goalpost:draft`, `/goalpost:review`,
`/goalpost:submit`) are Claude-Code-only — Codex's plugin spec
doesn't yet cover slash commands.

To skip the plugin layer entirely and wire Codex up by hand, paste
the snippet from [`codex/config.toml.example`](./codex/config.toml.example)
into `~/.codex/config.toml`.

### Claude Desktop

Open **Settings → Connectors**, choose **Add custom connector**, name
it Goalpost, and paste `https://api.movegoalpost.com/api/mcp` as the
address. OAuth runs on first use.

Older builds without the Connectors panel can use the `npx mcp-remote`
shim instead: edit
`~/Library/Application Support/Claude/claude_desktop_config.json`
(macOS) or `%APPDATA%\Claude\claude_desktop_config.json` (Windows),
add the block below, and restart Claude Desktop.

```json
{
  "mcpServers": {
    "goalpost": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "https://api.movegoalpost.com/api/mcp"]
    }
  }
}
```

### Cursor

Either click an install deeplink (see
[movegoalpost.com/docs/ai-integration](https://www.movegoalpost.com/docs/ai-integration))
or paste the same `mcp-remote` config into Cursor's MCP settings.

### Other clients

Most clients that don't yet speak remote MCP natively can use the
`npx mcp-remote` shim with the URL
`https://api.movegoalpost.com/api/mcp`. Native streamable-HTTP
clients can point at that URL directly — OAuth discovery follows the
standard RFC 9728 chain from the endpoint.

## What's in this plugin

- **MCP server** — points at `https://api.movegoalpost.com/api/mcp`.
  Twenty-four tools mirroring Goalpost's command layer
  (`list_projects`, `create_change_batch`, `assign_team_to_project`, …).
  Claude Code consumes it as a native streamable-HTTP server; Codex
  consumes the same server through an `npx mcp-remote` stdio shim.
- **Skill** — `skills/goalpost/SKILL.md`, loaded by both Claude Code
  and Codex. Teaches the workspace/project/change-batch/detail mental
  model, the detail-ID minting and operation-type conventions,
  in-place draft revision, and the habit of surfacing `webappUrl`
  back to the user.
- **Slash commands** — `/goalpost:draft`, `/goalpost:review`,
  `/goalpost:submit`. Short, opinionated workflows that compose
  multiple MCP tool calls into a single user intent.
  **Claude-Code-only**: Codex's plugin spec doesn't yet cover slash
  commands, so Codex users get the MCP server + skill but invoke the
  workflows by asking in natural language.

## Multiple workspaces

Goalpost OAuth tokens are bound to a single workspace at consent
time. If you belong to more than one Pro-tier workspace and want to
work in each via the assistant, repeat the OAuth flow per workspace
and register each as a separate MCP server entry.

> **Planned**: a `/settings/integrations` page in the Goalpost
> webapp will generate per-workspace install snippets so you don't
> have to construct them by hand. Not shipped yet.

## Disconnecting

Open **Profile → Connected apps** in the Goalpost webapp and revoke the
entry for your assistant. Its access ends immediately; removing the
plugin on the client side alone does not revoke anything server-side.

## Privacy and data handling

- The assistant only sees data inside the workspace this connection
  is bound to.
- Tokens are stored by your AI client locally and refreshed via the
  standard OAuth flow.
- When installed via this plugin (Claude Code, Claude Desktop, Codex,
  Cursor), tool calls go directly between your client and Goalpost;
  Anthropic / OpenAI / etc. do not proxy them. Note: a future
  Anthropic Connectors directory listing for Goalpost would route
  through Anthropic's infrastructure instead — that path isn't this
  plugin.

## Open questions / TODOs

- **Mirror to `movegoalpost/goalpost-plugin`.** The `marketplace.json`
  is already in place under `.claude-plugin/` (Codex reads that path
  for legacy compatibility); both `claude plugin marketplace add`
  and `codex plugin marketplace add` will resolve once the public
  repo exists. `npm run plugin:mirror -- --push` from the Goalpost
  repo publishes this directory into it as a snapshot commit (see
  `devops/README.md`); the public repo carries none of the private
  repo's history.
- **Drop the `mcp-remote` shim when Codex grows OAuth-on-first-use.**
  Codex supports remote streamable-HTTP MCP today but requires an
  explicit `codex mcp login <name>` to complete OAuth — no RFC 9728
  discovery on first tool call. Tracked in
  [openai/codex#23846](https://github.com/openai/codex/issues/23846);
  resolving that issue alone isn't enough. See
  [`codex/README.md`](./codex/README.md) for the conditions under
  which the shim can be removed.
- **`SKILL.md` ↔ MCP server `instructions` drift.** The skill's
  mental-model content duplicates the `MCP_INSTRUCTIONS` block in
  `api/src/routes/mcp.ts` by design (the skill persists across
  sessions, the instructions fire per connect), but the two will
  drift. Per
  [`docs/llm-harness-integration.md` §14](https://github.com/onahillco/goalpost/blob/main/docs/llm-harness-integration.md),
  a test should assert they agree — file in the Goalpost repo when
  this plugin moves to its public home.
- **Cursor `cursor://` install deeplink.** Generate and publish from
  `/settings/integrations`; document here as an alternative to the
  manual config.
- **Claude Code deeplink** (`claude://mcp/install?…`) as an
  alternative to the CLI invocation, once Anthropic stabilizes the
  URL scheme.

## Support

- Documentation: <https://www.movegoalpost.com/docs/ai-integration>
- Issues: <https://github.com/movegoalpost/goalpost-plugin/issues>
- Email: <info@onahill.co>
