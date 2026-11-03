# Goalpost plugin for AI assistants

Connect [Goalpost](https://www.movegoalpost.com) — shared, versioned
project documents for people and agents — to your AI assistant. Once installed, the
assistant can read your projects, draft and revise change batches, and
manage project-level team access on your behalf.

The published copy of this plugin lives at
[`movegoalpost/goalpost-plugin`](https://github.com/movegoalpost/goalpost-plugin);
changes made in the Goalpost monorepo reach it through
`npm run plugin:mirror`.

Works with every Goalpost workspace, Free or Pro.

## What you (or your assistant) can do

- List projects, read the latest approved version of a project, and browse
  change batch history.
- Create a new project.
- Create draft change batches, iterate on them, split, merge, or copy
  them, and submit them into the approval queue.
- Create and edit labels.
- List teams in the workspace and see which teams can access a project.
- As a workspace owner: create, rename, archive, and delete teams,
  add and remove their members, assign them to projects, and create
  and manage approval rules.

## What this connection cannot do

By design, the following actions are not exposed and must be done in
the Goalpost webapp:

- Cast approval votes (`approve`, `reject`, `withdraw`). The plugin
  *can* submit a draft for approval, but it can't cast votes on it.
- Team and approval rule changes when you are not a workspace owner.
- Invite people to the workspace or remove them from it.
- Delete projects or change batches.
- Upload new images (existing project assets can be referenced).
- Anything under workspace settings or billing.
- Authentication or token management.

## Install

Most people don't need this plugin. Claude Desktop and claude.ai add
Goalpost as a **custom connector**, and ChatGPT adds it as a **custom
MCP server**; both take the address
`https://api.movegoalpost.com/api/mcp` and run OAuth on first use.
The steps, with screenshots, are at
[movegoalpost.com/docs/ai-integration](https://www.movegoalpost.com/docs/ai-integration).

The plugin is for Claude Code and the Codex CLI, where it adds the
Goalpost Skill (and, in Claude Code, slash commands) on top of the
connection, and it is the bundle the assistant directories take.

### Claude Code

Install via the marketplace:

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

Without the plugin, `claude mcp add goalpost --transport
streamable-http https://api.movegoalpost.com/api/mcp` connects the
server alone.

### Codex CLI

Add the marketplace from a shell:

```bash
codex plugin marketplace add movegoalpost/goalpost-plugin
```

Then run `/plugins` inside a Codex session and install **Goalpost**.

Then sign in once, which opens Goalpost's consent screen in your
browser:

```bash
codex mcp login goalpost
```

The plugin bundles the same native streamable-HTTP server entry
Claude Code uses, and the Goalpost Skill rides along. Slash commands
(`/goalpost:draft`, `/goalpost:review`, `/goalpost:submit`) are
Claude-Code-only — the Codex plugin spec doesn't yet cover slash
commands.

To skip the plugin layer entirely and wire Codex up by hand, paste
the snippet from [`codex/config.toml.example`](./codex/config.toml.example)
into `~/.codex/config.toml` and run the same login command.

### Testing the marketplace in the desktop apps

The desktop apps can install the plugin from this repository too,
which is how to check a marketplace change before publishing:

- **Claude Desktop**: Settings → Plugins → + Add → Add marketplace →
  Add from a repository, enter `movegoalpost/goalpost-plugin`, Sync,
  then **Add to this computer** on the Goalpost row and **Connect**
  on its server.
- **ChatGPT desktop**: Settings → Plugins → Add → Add a marketplace,
  source `movegoalpost/goalpost-plugin`, Git ref `main`, no sparse
  paths, then install Goalpost from the **Marketplace** tab.

### Cursor

Either click an install deeplink (see
[movegoalpost.com/docs/ai-integration](https://www.movegoalpost.com/docs/ai-integration))
or add `https://api.movegoalpost.com/api/mcp` as a remote MCP server
in Cursor's MCP settings; OAuth runs on first use.

### Other clients

Native streamable-HTTP clients point at
`https://api.movegoalpost.com/api/mcp` directly — OAuth discovery
follows the standard RFC 9728 chain from the endpoint. For a client
that only speaks stdio, see the "Other assistants" section of
[movegoalpost.com/docs/ai-integration](https://www.movegoalpost.com/docs/ai-integration).

## What's in this plugin

- **MCP server** — points at `https://api.movegoalpost.com/api/mcp`.
  Tools mirroring Goalpost's command layer (`list_projects`,
  `create_change_batch`, `submit_for_approval`, …), plus team and
  approval rule management for workspace owners.
  Claude Code, Claude Desktop, and ChatGPT / Codex all consume it as
  a native streamable-HTTP server from the one `.mcp.json`.
- **Skill** — `skills/goalpost/SKILL.md`, loaded by Claude Code,
  Claude Desktop, and ChatGPT / Codex. Teaches the
  workspace/project/change-batch/detail mental model, the detail-ID minting and operation-type conventions,
  in-place draft revision, and the habit of surfacing `webappUrl`
  back to the user.
- **Slash commands** — `/goalpost:draft`, `/goalpost:review`,
  `/goalpost:submit`. Short, opinionated workflows that compose
  multiple MCP tool calls into a single user intent.
  **Claude-Code-only**: the Codex plugin spec doesn't yet cover slash
  commands, so ChatGPT users get the MCP server + skill but invoke the
  workflows by asking in natural language.

## Multiple workspaces

Goalpost OAuth tokens are bound to a single workspace at consent
time. If you belong to more than one workspace and want to
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
- When installed via this plugin (Claude Code, Claude Desktop, ChatGPT,
  Cursor), tool calls go directly between your client and Goalpost;
  Anthropic / OpenAI / etc. do not proxy them. Note: a future
  Anthropic Connectors directory listing for Goalpost would route
  through Anthropic's infrastructure instead — that path isn't this
  plugin.

## Open questions / TODOs

- **Sign-in on first tool call in Codex.** Codex completes OAuth for
  a remote server on `codex mcp login <name>` rather than on the first
  call; the marketplace's `ON_INSTALL` policy should prompt for it at
  install in the ChatGPT app, which is still to be confirmed by hand.
  Tracked in
  [openai/codex#23846](https://github.com/openai/codex/issues/23846);
  see [`codex/README.md`](./codex/README.md).
- **`SKILL.md` ↔ MCP server `instructions` drift.** The skill's
  mental-model content duplicates the `MCP_INSTRUCTIONS` block in
  `api/src/routes/mcp.ts` by design (the skill persists across
  sessions, the instructions fire per connect), but the two will
  drift. Per
  [`docs/llm-harness-integration.md` §14](https://github.com/onahillco/goalpost/blob/main/docs/llm-harness-integration.md),
  a test should assert they agree — to be filed in the Goalpost repo.
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
