---
description: Use Goalpost — shared, versioned project documents for people and agents — well when reading projects or authoring/revising change batches through the `goalpost` MCP server.
---

# Working in Goalpost

Goalpost keeps shared, versioned project documents for people and
agents. Each MCP connection is
pinned to a single **workspace**; every tool call reads and writes
inside that workspace. The workspace is implicit and is never passed
as an argument.

## Mental model

- **Project** — a named, structured document (a product spec, a meal
  plan, a garden layout, a running set of notes) living inside the
  workspace. Underneath, it is a tree of details whose nodes come in a
  few types: `SECTION`, `TEXT`, `LINK`, `IMAGE`, and `REF`, described
  below.
- **ChangeBatch** — a set of operations that edit a project's
  contents, anchored to a parent change batch. When a change batch is
  saved, Goalpost regenerates the resulting project state by applying
  its operations on top of the parent's state.
  - Lifecycle: `draft` (freely editable) → `pending_approval`
    (awaiting reviewers) → `approved` (permanent, sequentially
    numbered, immutable).
  - Approved change batches form a linear chain; exactly one is "latest"
    for a project at a time.
- **Detail** — a node in a project's hierarchical content tree,
  identified by `detail_<uuid>`. Details have a
  `parent_detail_id`, an `order_key` within their siblings, a `type`,
  and `content` whose shape depends on the type:
  - `TEXT` — a note: a requirement, a decision, a statement, anything
    written down; `content` is `{ "text": ... }`.
  - `SECTION` — a named grouping, like a folder; `content` is
    `{ "title": ... }`.
  - `LINK` — an http(s) link to something outside the project;
    `content` is `{ "url": ..., "label"?: ... }`. Placed like `TEXT`.
  - `REF` — a reference to another detail in the same project;
    `content` is `{ "detail_id": ... }` and the reference reads as its
    target. Placed like `TEXT`. The target must exist in every version the reference exists
    in: remove or retarget references before removing their target.
    Replacing a target carries its references to the replacement.
  - `IMAGE` — an image uploaded into the project; `content` is
    `{ "asset_id": ..., "alt": ... }`. Reference an asset that already
    exists: `list_project_assets` gives you the ids for a project and
    `get_project_asset_url` returns a short-lived URL to look at one.
    `alt` is required. Placed like `TEXT`. **You cannot upload image
    bytes over this connection** — if the user wants a new picture in
    the project, they add it in the Goalpost webapp, and you can reference
    it afterwards.
- **Sections are the spine of the tree.** Every root detail must be a
  `SECTION`, and a `SECTION` may only sit under another `SECTION`.
  `TEXT` details live inside sections (and may nest under other `TEXT`
  details). To add content to an empty project, add a root `SECTION`
  first and put `TEXT` details under it. Moving a `TEXT` detail to the
  root, or a `SECTION` under a `TEXT` detail, is rejected.
- **Operation** — a tree-edit primitive carried by a change batch. Four
  kinds, each with a specific job:
  - `ADD_DETAIL` — introduce a new detail.
  - `REMOVE_DETAIL` — delete a detail and all its descendants.
  - `MOVE_DETAIL` — change a detail's `parent_detail_id` and/or
    `position` in place. The id, content, and children are preserved.
  - `REPLACE_DETAIL` — swap a detail for a new node with a new id.
    Scoped to content edits: type, parent, and position are inherited
    from the predecessor (the replacement must carry the same `type`),
    and the predecessor's children are re-parented to the replacement.

## Typical workflow

1. `list_projects` to discover what exists.
2. `view_change_batch` when the user asks to see, show, open, look at
   or review a project, a draft or a pending change. It opens the
   project beside the conversation (where the host renders it)
   and returns its state, so you don't need a second read. Pass
   `changeBatchId` for a draft or pending change; without it the latest
   approved version opens. `get_latest_approved_change_batch` (or
   `get_change_batch` for a specific one) reads the same state without
   showing it — use those when you need the state for your own work
   and the user hasn't asked to see it.
3. `create_change_batch` to start a draft. The server defaults
   `parentId` to the latest approved change batch; pass it explicitly
   only when branching from an older one.
4. `update_change_batch` to iterate on the draft's operations.
5. `submit_for_approval` to send the draft into the approval
   workflow.

`list_teams` / `list_project_teams` read which teams exist and which
can access a project. Any user can leave a team with
`remove_team_member`, passing their own `userId` from
`get_current_workspace`.

### Workspace owners: teams and approval rules

When the connected user owns the workspace (`get_current_workspace`
reports `role: "owner"`), the server also registers tools to manage
teams and approval rules: `create_team`, `rename_team`, `delete_team`,
`set_team_archived`, `set_team_approval_rule`, `add_team_member`,
`assign_team_to_project`,
`remove_team_from_project`, `create_approval_rule`,
`update_approval_rule`, `set_approval_rule_archived`,
`delete_approval_rule`, `set_project_approval_rule`, and the reads
`get_team`, `list_team_projects`, `list_workspace_members`,
`list_workspace_approval_rules`, `get_project_approval_rule`, and
`list_approval_rule_projects`. Members don't get these tools; route
them to a workspace owner.

These change who can see projects and who must approve every future
change, immediately, outside any draft. Confirm with the user before
calling a mutating one. Resolve a person's `userId` with
`list_workspace_members` before adding them to or removing them from a
team.

For images, `list_project_assets` and `get_project_asset_url` read a
project's asset store. Uploading is webapp-only.

## Minting and threading detail IDs

Detail IDs are **client-generated**. You — not the server — produce
them. The rules:

- Mint a fresh `detail_<uuid>` whenever an operation introduces a
  detail. That's `ADD_DETAIL.id` and `REPLACE_DETAIL.id`. The `<uuid>`
  is a freshly-generated v4 UUID; do not reuse an old one and do not
  invent a short or human-readable name.
- `MOVE_DETAIL` reuses the existing detail's id — it relocates a node
  rather than minting a new one.
- For any child you add in the same operations array, reuse the id
  you just minted as that child's `parent_detail_id`. Don't fetch the
  draft back to look it up.
- For details inherited from the approved baseline, use the id
  returned by `get_change_batch` or `get_latest_approved_change_batch`.

## Choosing the right operation type

The choice between operation types matters because it changes what
happens to a detail's identity and its descendants. Default toward
the operation that preserves more of the existing tree:

- **Editing a detail's content** — use `REPLACE_DETAIL`, not
  `REMOVE + ADD`. REPLACE re-parents the predecessor's children to
  the replacement (descendants survive); REMOVE_DETAIL deletes them.
  A REMOVE+ADD will silently destroy referenced state.
- **Moving a detail to a new parent or position** — use
  `MOVE_DETAIL`, not `REMOVE + ADD`. MOVE preserves the detail's id,
  which is the right signal to anyone reading history that this is
  "the same thing, in a new place." REMOVE+ADD makes it look like the
  old detail was deleted and an unrelated new one was created.
- **Editing AND moving in the same change batch** — append both ops.
  But remember: after `REPLACE_DETAIL`, the predecessor's id is gone.
  Any subsequent MOVE/REPLACE/REMOVE in the same op array must target
  the new id.

## Let the tree carry the structure

The tree *is* the outline. Sibling order comes from `order_key`, and
the webapp renders nesting and order on its own. Don't duplicate that
structure in the content:

- **No outline numbering in titles or text.** Write `Offerings`, not
  `3. Offerings`; `Storage Services`, not `3.1 Storage Services`; the
  same goes for `A)`, `(ii)`, `Step 4:` and similar prefixes. Baked-in
  numbers go stale the moment a detail is inserted, moved, or removed,
  and fixing them takes a `REPLACE_DETAIL` on every later sibling —
  a noisy diff reviewers have to wade through, and churned history
  for content that didn't change. To reorder, `MOVE_DETAIL` (or pick
  a new `order_key`) and nothing else needs to change.
- **Strip numbers when transcribing.** If the user hands you a
  numbered document, drop the numbering as you turn it into details.
  Don't add numbering of your own to a project. Goalpost has first-class
  handling for ordering content, and renumbering sections or other
  lists later is expensive.
- **Numbers that mean something stay.** `Phase 2`, `Tier 3 support`,
  or an external standard's clause id (`ISO 27001 A.5.1`) are content,
  not position — keep them.
- **Cross-reference with `REF`, not by number.** Instead of "see
  section 3.2", add a `REF` detail pointing at the target. It reads as
  the target, survives reordering, and follows the target through a
  `REPLACE_DETAIL`.
- **Nest instead of formatting.** Don't encode hierarchy inside a
  `TEXT` detail with bullets, dashes, or indentation; give each item
  its own child detail. One requirement per detail keeps edits,
  labels, and approval rules precise.

## Revising a draft in place

A draft is a mutable workspace, not an append-only log. When the user
asks for changes to a draft you already wrote:

- For a detail this draft introduced via `ADD_DETAIL` — **edit that
  ADD_DETAIL operation in place** in the operations array. Don't
  append a follow-up REPLACE or MOVE op on top of it. Use
  `update_change_batch` with the revised operations array.
- For a detail inherited from the approved baseline — append a
  `REPLACE_DETAIL` (for content) or `MOVE_DETAIL` (for
  relocation), per the rules above. You can't edit baseline details
  in place; you have to record an operation against them.

This keeps the change batch's operations list compact and tells a clean
story for reviewers.

## After a tool call

When the user asked to see something and you called
`view_change_batch`, the opened view is the answer where the host
renders it: summarize briefly and don't restate the project's contents
in your reply.

When a tool response includes a `webappUrl` field, **surface it to
the user in your reply** — it's a clickable bridge to the
Goalpost webapp where they can verify your work in the real UI.
Tools that mutate or return a specific project or change batch carry
this field (create/update/submit on change batches, create_project,
assign_team_to_project, create_team, the approval rule create/update
and project assignment, and the change batch reads). List operations
and tools that return nothing don't.

## What you cannot do via this connection

The MCP surface is intentionally scoped to reading and drafting, plus
team and approval rule management for workspace owners. These are
**out of scope** — route the user to the Goalpost webapp instead:

- Casting approval votes (`approve` / `reject` / `withdraw`).
- Team and approval rule management, when the user is not a
  workspace owner (an owner has to make the change).
- Inviting people to or removing them from the workspace.
- Anything under workspace settings or billing.
- Authentication or token management.
- Uploading image bytes. You can reference and view assets that are
  already in the project, but adding a new one happens in the webapp.

If the user asks for one of these, explain that it has to happen in
the webapp and (if relevant) include the `webappUrl` from the most
recent tool response so they have a starting point.
