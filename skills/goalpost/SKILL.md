---
name: goalpost
description: Use Goalpost — shared, versioned documents for people and agents — well when reading documents or authoring/revising change batches through the `goalpost` MCP server.
---

# Working in Goalpost

Goalpost keeps shared, versioned documents for people and
agents. Each MCP connection is
pinned to a single **workspace**; every tool call reads and writes
inside that workspace. The workspace is implicit and is never passed
as an argument.

## Mental model

- **Document** — a named, structured body of content (a product spec, a meal
  plan, a garden layout, a running set of notes) living inside the
  workspace. Underneath, it is a tree of items whose nodes come in a
  few types: `SECTION`, `TEXT`, `LINK`, `IMAGE`, and `REF`, described
  below.
- **ChangeBatch** — a set of operations that edit a document's
  contents, anchored to a parent change batch. When a change batch is
  saved, Goalpost regenerates the resulting document state by applying
  its operations on top of the parent's state.
  - Lifecycle: `draft` (freely editable) → `pending_approval`
    (awaiting reviewers) → `approved` (permanent, sequentially
    numbered, immutable).
  - Approved change batches form a linear chain; exactly one is "latest"
    for a document at a time.
- **Item** — a node in a document's hierarchical content tree: every
  section, note, link, image, and reference in the document is an item,
  identified by `item_<uuid>`. When you tell the user what you did,
  name items by type where you can ("added a note under Requirements",
  "moved the Budget section") and keep "item" for the general case.
  Items have a
  `parent_item_id`, an `order_key` within their siblings (read-only:
  the server mints it when an item is placed; never write one), a
  `type`, and `content` whose shape depends on the type:
  - `TEXT` — a note: a requirement, a decision, a statement, anything
    written down; `content` is `{ "text": ... }`.
  - `SECTION` — a named grouping, like a folder; `content` is
    `{ "title": ... }`.
  - `LINK` — an http(s) link to something outside the document;
    `content` is `{ "url": ..., "label"?: ... }`. Placed like `TEXT`.
  - `REF` — a reference to another item in the same document;
    `content` is `{ "item_id": ... }` and the reference reads as its
    target. Placed like `TEXT`. The target must exist in every version the reference exists
    in: remove or retarget references before removing their target.
    Replacing a target carries its references to the replacement.
  - `IMAGE` — an image uploaded into the document; `content` is
    `{ "asset_id": ..., "alt": ... }`. Reference an asset that already
    exists: `list_document_assets` gives you the ids for a document and
    `get_document_asset_url` returns a short-lived URL to look at one.
    `alt` is required. Placed like `TEXT`. **You cannot upload image
    bytes over this connection** — if the user wants a new picture in
    the document, they add it in the Goalpost webapp, and you can reference
    it afterwards.
- **Sections are the spine of the tree.** Every root item must be a
  `SECTION`, and a `SECTION` may only sit under another `SECTION`.
  Every other item sits under a section or under any other item, so a
  `TEXT` item can carry `TEXT`, `LINK` and `REF` items of its own. To add
  content to an empty document, add a root `SECTION` first and put
  `TEXT` items under it. Moving a `TEXT` item to the root, or a
  `SECTION` under a `TEXT` item, is rejected.
- **Operation** — a tree-edit primitive carried by a change batch. Four
  kinds, each with a specific job:
  - `ADD_ITEM` — introduce a new item, placed by `parent_item_id`
    (null for the root) and, when it is not simply last under that
    parent, by `after_item_id` or `before_item_id` naming the sibling
    it goes next to.
  - `REMOVE_ITEM` — delete an item and all its descendants.
  - `MOVE_ITEM` — change an item's parent and place, named the same
    way. The id, content, and children are preserved.
  - `REPLACE_ITEM` — swap an item for a new node with a new id.
    Scoped to content edits: type, parent, and place are inherited
    from the predecessor (the replacement must carry the same `type`),
    and the predecessor's children are re-parented to the replacement.

## Typical workflow

1. `list_documents` to discover what exists.
2. `view_change_batch` when the user asks to see, show, open, look at
   or review a document, a draft or a pending change. It opens the
   document beside the conversation (where the host renders it)
   and returns its state, so you don't need a second read. Pass
   `changeBatchId` for a draft or pending change; without it the latest
   approved version opens. Pass `changesOnly: true` to open on the list
   of what that version changed rather than the whole document — the
   view for reviewing a draft or a pending change. `get_latest_approved_change_batch` (or
   `get_change_batch` for a specific one) reads the same state without
   showing it — use those when you need the state for your own work
   and the user hasn't asked to see it.
3. `create_change_batch` to start a draft. The server defaults
   `parentId` to the latest approved change batch; pass it explicitly
   only when branching from an older one. To write a whole tree, list
   the items in the order they should appear, each after its parent,
   and name no sibling: each lands last under its parent as it goes.
4. `append_change_batch_operations` to add to the draft (a subtree, a
   few edits) without resending what it has; the single-item tools
   (`add_item_to_change_batch` and friends) for one change at a time;
   `remove_change_batch_operation` to take back one change the draft
   made to an item from the approved version.
5. `submit_for_approval` to send the draft into the approval
   workflow.

`list_teams` / `list_document_teams` read which teams exist and which
can access a document. Any user can leave a team with
`remove_team_member`, passing their own `userId` from
`get_current_workspace`.

### Workspace owners: teams and approval rules

When the connected user owns the workspace (`get_current_workspace`
reports `role: "owner"`), the server also registers tools to manage
teams and approval rules: `create_team`, `rename_team`, `delete_team`,
`set_team_archived`, `set_team_approval_rule`, `add_team_member`,
`assign_team_to_document`,
`remove_team_from_document`, `create_approval_rule`,
`update_approval_rule`, `set_approval_rule_archived`,
`delete_approval_rule`, `set_document_approval_rule`, and the reads
`get_team`, `list_team_documents`, `list_workspace_members`,
`list_workspace_approval_rules`, `get_document_approval_rule`, and
`list_approval_rule_documents`. Members don't get these tools; route
them to a workspace owner.

These change who can see documents and who must approve every future
change, immediately, outside any draft. Confirm with the user before
calling a mutating one. Resolve a person's `userId` with
`list_workspace_members` before adding them to or removing them from a
team.

For images, `list_document_assets` and `get_document_asset_url` read a
document's asset store. Uploading is webapp-only.

## Minting and threading item IDs

Item IDs are **client-generated**. You — not the server — produce
them. The rules:

- Mint a fresh `item_<uuid>` whenever an operation introduces an
  item. That's `ADD_ITEM.id` and `REPLACE_ITEM.id`. The `<uuid>`
  is a freshly-generated v4 UUID; do not reuse an old one and do not
  invent a short or human-readable name.
- `MOVE_ITEM` reuses the existing item's id — it relocates a node
  rather than minting a new one.
- For any child you add in the same operations array, reuse the id
  you just minted as that child's `parent_item_id`. Don't fetch the
  draft back to look it up.
- For items inherited from the approved baseline, use the id
  returned by `get_change_batch` or `get_latest_approved_change_batch`.

## Choosing the right operation type

The choice between operation types matters because it changes what
happens to an item's identity and its descendants. Default toward
the operation that preserves more of the existing tree:

- **Editing an item's content** — use `REPLACE_ITEM`, not
  `REMOVE + ADD`. REPLACE re-parents the predecessor's children to
  the replacement (descendants survive); REMOVE_ITEM deletes them.
  A REMOVE+ADD will silently destroy referenced state.
- **Moving an item to a new parent or position** — use
  `MOVE_ITEM`, not `REMOVE + ADD`. MOVE preserves the item's id,
  which is the right signal to anyone reading history that this is
  "the same thing, in a new place." REMOVE+ADD makes it look like the
  old item was deleted and an unrelated new one was created.
- **Editing AND moving in the same change batch** — append both ops.
  But remember: after `REPLACE_ITEM`, the predecessor's id is gone.
  Any subsequent MOVE/REPLACE/REMOVE in the same op array must target
  the new id.

## Let the tree carry the structure

The tree *is* the document. Sibling order comes from where operations
place items, and the webapp renders nesting and order on its own. Don't
duplicate that structure in the content:

- **No section numbering in titles or text.** Write `Offerings`, not
  `3. Offerings`; `Storage Services`, not `3.1 Storage Services`; the
  same goes for `A)`, `(ii)`, `Step 4:` and similar prefixes. Baked-in
  numbers go stale the moment an item is inserted, moved, or removed,
  and fixing them takes a `REPLACE_ITEM` on every later sibling —
  a noisy diff reviewers have to wade through, and churned history
  for content that didn't change. To reorder, `MOVE_ITEM` and
  nothing else needs to change.
- **Strip numbers when transcribing.** If the user hands you a
  numbered document, drop the numbering as you turn it into items.
  Don't add numbering of your own to a document. Goalpost has first-class
  handling for ordering content, and renumbering sections or other
  lists later is expensive.
- **Numbers that mean something stay.** `Phase 2`, `Tier 3 support`,
  or an external standard's clause id (`ISO 27001 A.5.1`) are content,
  not position — keep them.
- **Cross-reference with `REF`, not by number.** Instead of "see
  section 3.2", add a `REF` item pointing at the target. It reads as
  the target, survives reordering, and follows the target through a
  `REPLACE_ITEM`.
- **Prefer one point per item, with its detail beneath it.** A
  paragraph in one `TEXT` item is allowed, but don't default to it.
  Where text makes several points, give each its own item, and put what
  elaborates a point (the reason, an exception, an example, a
  definition, its source) in child items under it rather than in more
  sentences of the same text. A list reads best as one item per entry
  rather than bullets or dashes inside one text. Small items can each be
  edited, labeled, referenced, approved, and compared on their own,
  while a change to one sentence of a long paragraph replaces the whole
  paragraph and makes reviewers read all of it again. A paragraph suits
  prose that makes one point, such as a short overview or a quotation.

## Revising a draft in place

A draft holds the net effect of its edits, not a log of them, and the
server keeps it that way. When the user asks for changes to a draft you
already wrote, make each change with the item tools
(`edit_item_in_change_batch`, `move_item_in_change_batch`,
`remove_item_from_change_batch`) or `append_change_batch_operations`:

- An edit, move or removal of an item this draft added is folded into
  its ADD_ITEM, so the draft still shows one addition.
- An edit or move of an item from the approved version is recorded
  against it, replacing any earlier edit of the same item.
- To take back a change to an item from the approved version (keep an
  item the draft removed, restore its wording or its place), read the
  draft with `get_change_batch` and call `remove_change_batch_operation`
  with that change's operation `id` and the read's
  `current_checkpoint_id`. The draft's other changes stay.

This keeps the change batch's operations list compact and tells a clean
story for reviewers.

## After a tool call

When the user asked to see something and you called
`view_change_batch`, the opened view is the answer where the host
renders it: summarize briefly and don't restate the document's contents
in your reply.

When a tool response includes a `webappUrl` field, **surface it to
the user in your reply** — it's a clickable bridge to the
Goalpost webapp where they can verify your work in the real UI.
Tools that mutate or return a specific document or change batch carry
this field (create/update/submit on change batches, create_document,
assign_team_to_document, create_team, the approval rule create/update
and document assignment, and the change batch reads). List operations
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
  already in the document, but adding a new one happens in the webapp.

If the user asks for one of these, explain that it has to happen in
the webapp and (if relevant) include the `webappUrl` from the most
recent tool response so they have a starting point.
