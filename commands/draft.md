---
description: Draft a new Goalpost change batch from the current conversation. Picks a project, reads its latest approved state, and proposes an operations array for review before saving.
---

# Goalpost — draft a change batch

The user wants to turn the current conversation into a Goalpost
change batch. Walk them through it without rushing to save:

1. **Confirm the workspace and project.** Call
   `get_current_workspace` once so you can name the workspace in your
   reply. Then `list_projects` and ask the user to pick the project
   this draft belongs in — unless the conversation has already named
   one unambiguously. If the user wants a brand-new project, call
   `create_project` first and surface its `webappUrl`.

2. **Read the baseline.** Call `get_latest_approved_change_batch` for the
   chosen project. If it returns a change batch, that's your reference
   for any existing details you'll edit or move. If it returns
   `null`, this is the project's first change batch — every detail will
   be an `ADD_DETAIL`.

3. **Propose the operations array first, don't save it yet.** Before
   calling `create_change_batch`, write out the operations you intend to
   include in a numbered list in your reply. Group them logically
   (new sections, then edits, then moves, then removals). Use the
   convention from the goalpost skill: `REPLACE_DETAIL` for content
   edits to baseline details, `MOVE_DETAIL` for relocations,
   `ADD_DETAIL` for new content, `REMOVE_DETAIL` only when the user
   has explicitly asked for a deletion. Mint fresh `detail_<uuid>`
   ids for ADD/REPLACE.

4. **Ask the user to confirm or revise.** Wait for their go-ahead on
   the operations list before saving. If they want changes, revise
   the list in-place; don't accumulate alternatives.

5. **Save the draft.** Call `create_change_batch` with the agreed
   operations array. Surface the returned `webappUrl` so the user
   can review the rendered specification in the webapp. Mention that
   the draft is editable via `update_change_batch` and gets sent for
   review via `submit_for_approval`.

If at any point a tool returns a validation error (bad parent_detail_id,
duplicate id, etc.), explain what went wrong in plain language, fix
the operations array, and try again.
