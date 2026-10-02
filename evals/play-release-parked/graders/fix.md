---
type: llm
weight: 1
focus: last_message
---
Context: a Play Developer API script commits with `changesNotSentForReview=True` first and falls back to a plain commit only on an error. The flagged commit succeeds, so the production release sits in Console without being sent for review. The developer also asks what the script should check at the end.

PASS if the answer does both:
1. The default path commits WITHOUT `changesNotSentForReview` (a plain `edits.commit`, or the flag set to false). The flag is used only if the API rejects a plain commit and asks for it, and then a human must press "Send changes for review" in Play Console.
2. It covers the stuck release or the proof, with at least one of these: send the pending 412 changes for review in Play Console now; or, after the commit, read the production track back (a new edit or a separate read, ideally with a read-only service account) and fail unless it shows the new versionCode with `status: completed`.

FAIL if any of these is true:
- It keeps trying the flagged commit first.
- It changes only the release status, rollout fraction or track, or adds a wait.
- Its "check" is only reading the track inside the same edit before the commit.
- It adds harmful advice, such as putting the upload service account key on a server, or deleting and re-creating the app listing.
