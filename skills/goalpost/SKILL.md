---
description: Use Goalpost — versioned specification documents — well when reading projects or authoring/revising change batches through the `goalpost` MCP server.
---

# Working in Goalpost

Goalpost is a versioned specification system. Each MCP connection is
pinned to a single **workspace**; every tool call reads and writes
inside that workspace. The workspace is implicit and is never passed
as an argument.

## Mental model

- **Project** — a named specification (a product spec, a meal plan, a
  garden layout, a set of guidelines) living inside the workspace.
- **ChangeBatch** — a set of operations that edit a project's
  specification, anchored to a parent change batch. When a change batch is
  saved, Goalpost regenerates the resulting specification by applying
  its operations on top of the parent's state.
  - Lifecycle: `draft` (freely editable) → `pending_approval`
    (awaiting reviewers) → `approved` (permanent, sequentially
    numbered, immutable).
  - Approved change batches form a linear chain; exactly one is "latest"
    for a project at a time.
- **Detail** — a node in the hierarchical content tree of a
  specification, identified by `detail_<uuid>`. Details have a
  `parent_detail_id`, an `order_key` within their siblings, a `type`,
  and `content` whose shape depends on the type:
  - `TEXT` — a requirement or statement; `content` is `{ "text": ... }`.
  - `SECTION` — a named grouping, like a folder; `content` is
    `{ "title": ... }`.
  - `LINK` — an http(s) link to something outside the specification;
    `content` is `{ "url": ..., "label"?: ... }`. Placed like `TEXT`.
  - `REF` — a reference to another detail in the same specification;
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
    the spec, they add it in the Goalpost webapp, and you can reference
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
2. `get_latest_approved_change_batch` (or `get_change_batch` for a specific
   one) to read the canonical state of a project.
3. `create_change_batch` to start a draft. The server defaults
   `parentId` to the latest approved change batch; pass it explicitly
   only when branching from an older one.
4. `update_change_batch` to iterate on the draft's operations.
5. `submit_for_approval` to send the draft into the approval
   workflow.

For team access management, use `list_teams` / `list_project_teams` to
read, and `assign_team_to_project` / `remove_team_from_project` to
write. Team creation, deletion, and membership changes are out of
scope for the MCP surface — direct the user to the Goalpost webapp
for those.

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

When a tool response includes a `webappUrl` field, **surface it to
the user in your reply** — it's a clickable bridge to the
Goalpost webapp where they can verify your work in the real UI.
Tools that mutate or return a specific project or change batch carry
this field (create/update/submit on change batches, create_project,
assign_team_to_project, and the change batch reads). List operations
and tools that return nothing don't.

## What you cannot do via this connection

The MCP surface is intentionally scoped to reading, drafting, and
project-level team access. These are **out of scope** — route the
user to the Goalpost webapp instead:

- Casting approval votes (`approve` / `reject` / `withdraw`).
- Configuring approval rules.
- Creating, renaming, or deleting teams.
- Adding or removing team members.
- Anything under workspace settings or billing.
- Authentication or token management.
- Uploading image bytes. You can reference and view assets that are
  already in the project, but adding a new one happens in the webapp.

If the user asks for one of these, explain that it has to happen in
the webapp and (if relevant) include the `webappUrl` from the most
recent tool response so they have a starting point.
