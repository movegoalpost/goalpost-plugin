---
description: "Send a Goalpost draft change batch into the approval queue: confirm it is still a draft, check its parent is the latest approved version, summarize it, and get explicit confirmation before calling submit_for_approval. Use when the user asks to submit a draft or send it for review."
---

# Goalpost — submit a draft for approval

The user is ready to send a draft into Goalpost's approval workflow.
Don't fire `submit_for_approval` immediately — confirm the draft is
in the shape they actually want first.

1. **Locate the draft.** If the user named one, call `get_change_batch`
   directly. Otherwise call `list_change_batches` for the document (ask
   which document if it isn't clear) and offer the most recent `draft`
   as the candidate.

2. **Verify it's still a draft.** If the change batch's status is
   already `pending_approval` or `approved`, tell the user and stop
   — `submit_for_approval` is only valid for drafts.

3. **Check the parent is current.** Call
   `get_latest_approved_change_batch` for the document and compare its id
   to the draft's `parentId`. If they differ (or `needsRebase` is set
   on the draft), the baseline has moved since the draft was written:
   - Explain that submitting now will queue the draft against a stale
     baseline.
   - Offer to reconcile the draft by reading the new baseline and
     editing the draft's operations array via `update_change_batch`
     before submitting — don't do this silently.
   - Wait for the user's call before continuing.

4. **Give the user a one-line summary of what they're about to
   submit.** "N operations across <document name> — adding X,
   replacing Y, moving Z." If the change batch has a name/description,
   include those. Ask for explicit confirmation.

5. **Submit.** Call `submit_for_approval` with the draft's id. Surface
   the returned `webappUrl` so the user can watch for reviewer
   activity in the webapp. Remind them — once — that approval votes
   themselves are cast in the webapp, not through this connection.
