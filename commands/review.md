---
description: Review a Goalpost change batch — summarize its operations and their consequences before it goes (or while it sits) in the approval queue.
---

# Goalpost — review a change batch

The user wants a human-readable summary of a change batch's contents and
the consequences of approving it. The change batch might be their own
draft they're about to submit, their own `pending_approval` waiting
for reviewers, or someone else's pending one they're voting on.

1. **Find and open the change batch.** If the user named a specific
   one, call `view_change_batch` with its id: it opens the change
   batch beside the conversation where the host renders it, and returns
   the same data `get_change_batch` would. Otherwise call
   `list_change_batches` for the document (ask them which document if it
   isn't clear), offer the most recent `draft` or `pending_approval` as
   the candidate — or the one they ask for — and open that.

2. **Check whether the draft is current.** The opened change batch
   carries a `needsRebase` field — if it's true, the
   approved baseline has moved since the draft was written and
   reconciling should happen before approval. Trust the field rather
   than re-deriving it. To inspect the new baseline, call
   `get_latest_approved_change_batch` for the document.

3. **Summarize per operation.** For each operation in the change batch,
   write one line in plain language:
   - `ADD_DETAIL` — describe the new content and where it lands
     (parent and position).
   - `REMOVE_DETAIL` — name the detail and call out that all its
     descendants go with it.
   - `MOVE_DETAIL` — name the detail and describe the move (from
     where, to where).
   - `REPLACE_DETAIL` — describe the content change and call out
     that the predecessor's children are re-parented to the
     replacement (descendants survive).

4. **Call out anything that warrants extra attention.** Examples:
   bulk removals, replacements affecting many descendants, edits to
   details referenced from elsewhere, or operations whose intent the
   conversation doesn't make clear. Surface these as a short
   "consider before approving" list.

5. **Surface the webapp link.** Include the `webappUrl` from the
   opened change batch so the user can verify against the rendered
   document before approving (or asking the author for
   changes). Remember: approving/rejecting/withdrawing votes are not
   available through this connection — direct the user to the webapp
   if they want to cast a vote.
