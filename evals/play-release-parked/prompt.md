---
name: play-release-parked
tags: [kmp-store-release]
runs: 2
max_turns: 6
timeout_seconds: 300
allowed_tools: [Read, Glob, Grep, Skill]
description: "A Play Developer API script commits with changesNotSentForReview=True first; the call succeeds, the fallback never runs, and the production release never reaches review."
expected_outcome: "Explains that the flagged commit succeeds and parks the changes without sending them for review, commits without the flag by default, sends the stuck release for review in Play Console, and reads the track back to prove it."
---
We moved our Android releases from clicking in Play Console to a Python script (Play Developer API v3, google-api-python-client). Our KMP travel app (Kotlin 2.4.10, CMP 1.11.1, AGP 8.13.2) shipped 3.4.0, versionCode 412, with it six days ago.

The script exits 0 and prints `released 412`. In Play Console, 412 is listed on the production track. But the store still serves 411 to everyone, we got no review email, and nothing looks like it is in review. Releases we made by hand in Console went live within a day. Managed publishing is off, and the service account has the release permissions.

```python
edit = svc.edits().insert(packageName=PKG, body={}).execute()["id"]
bundle = svc.edits().bundles().upload(
    packageName=PKG, editId=edit, media_body=MediaFileUpload(AAB)).execute()
code = bundle["versionCode"]
svc.edits().tracks().update(packageName=PKG, editId=edit, track="production", body={
    "track": "production",
    "releases": [{"versionCodes": [str(code)], "status": "completed",
                  "releaseNotes": [{"language": "en-US", "text": NOTES}]}]}).execute()
try:
    # needed when an app uses managed publishing, harmless otherwise
    svc.edits().commit(packageName=PKG, editId=edit, changesNotSentForReview=True).execute()
except HttpError:
    svc.edits().commit(packageName=PKG, editId=edit).execute()
print("released", code)
```

Is Google review just slow this week, or is the script wrong? And what should the script check at the end, so that "exit 0" really means the release went out?
