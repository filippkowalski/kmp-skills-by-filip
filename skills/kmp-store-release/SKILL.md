---
name: kmp-store-release
description: Google Play and App Store Connect behaviour a Kotlin Multiplatform team meets when one codebase ships to both stores. Use when you configure release builds, bump versions, upload or commit a Play release, set country availability, create store service accounts, roll out attestation, submit subscriptions for review, fill the age rating of an AI app, or build a force-update check. Triggers - applicationIdSuffix, preReleaseBuild, versionCode, CFBundleVersion, MARKETING_VERSION, ITMS-90062, ITMS-90186, 409, Play Developer API, edits.commit, changesNotSentForReview, tracks.get, images.upload, countryavailability, countryTargeting, service account, View app information, Play App Signing, assetlinks, Play Integrity, App Attest, App Check enforce, review submission, 2.1(b), age rating, messagingAndChat, 5.1.2, force update, minimum version, iTunes lookup.
---

# Shipping one KMP codebase to both stores

The code is shared; the stores are not. These are store and build-tool behaviours that decide
whether a release ships.

## 1. A debug `applicationIdSuffix` hides Play Billing products and breaks static shortcuts

- **Symptom:** Debug builds never see any Play Billing product. A launcher shortcut in the debug
  build opens nothing or the wrong app.
- **Cause:** `applicationIdSuffix` changes the installed package. Play Billing resolves products
  against the installed package name. A static shortcut in `shortcuts.xml` names an explicit
  component whose package is the installed application id.
- **Fix:** Keep the suffix if you need separate installs or separate data (many apps do), and
  generate `shortcuts.xml` per variant. Test billing on a build with the release id: real test
  purchases need a license-tester account and a signed build installed from a Play testing track.
  RevenueCat's testing guide: "A sideloaded debug build with a debug signing key cannot purchase
  licensed products, even for license testers." Dropping the suffix fixes the shortcuts, not
  sideloaded purchases, and costs one shared install, data store and Firebase app; swapping
  needs an uninstall because the signing keys differ. Mark such debug builds with `versionNameSuffix`.
- **Seen on:** Android Gradle Plugin 8.13, Play Billing through purchases-kmp 3.5.

See also: revenuecat-testing-setup in RevenueCat/ai-toolkit (license testers, the Internal Testing track, one dashboard entry per application id).

## 2. A blank release secret compiles fine: fail the release build yourself

- **Symptom:** A release build missing a required secret (an SDK key, an API key) builds, uploads
  and fails only at runtime.
- **Cause:** A `buildConfigField` fed from `local.properties` or an env var is an empty string
  when the value is missing. Gradle does not care.
- **Fix:** Fail `preReleaseBuild` when any required release secret is blank. Let debug stay
  unconfigured so a fresh clone compiles. Pin debug-only secrets to `""` in the release build type
  and read them only when the debug flag is true.

```kotlin
tasks.matching { it.name == "preReleaseBuild" }.configureEach {
    doFirst { if (requiredKey.isBlank()) throw GradleException("release needs sdk.releaseKey") }
}
```

- **Seen on:** Android Gradle Plugin 8.13. Purchases setup details: see kmp-revenuecat.

## 3. Each store has its own version rules

- **Symptom:** An iOS upload dies with ITMS-90062 or ITMS-90186. Submitting an older build returns
  409. An Android upload is refused as a duplicate.
- **Cause:**
  - Play: every upload needs a higher `versionCode`. The version name is free text.
  - App Store Connect: a build number is unique across all versions of the app. A build rejected
    after upload keeps its number. An upload refused at validation is not stored, so its number
    stays free.
  - An approved marketing version closes its train: uploads and submissions for it are refused.
    A version that is rejected or in review stays open.
- **Fix:** On Android bump `versionCode` and commit that change alone. On iOS read Apple's highest
  build number across all versions, add one, and stop if the read fails. Keep the marketing
  version above every approved one; a resubmission after a rejection reuses the open version with
  a new build. The two platforms' numbers will drift, so never compare versions across platforms.
  The read below uses `app-store-connect` from Codemagic CLI tools; `--all-versions` returns the
  highest build across every version, not the build of the highest version.

```sh
LATEST=$(app-store-connect get-latest-build-number --all-versions "$APP_ID") || exit 1
agvtool new-version -all $((LATEST + 1))
```

- **Seen on:** App Store Connect API, Play Developer API v3, Xcode agvtool, Codemagic CLI tools
  (`app-store-connect`).

## 4. If you publish through the Play Developer API, `changesNotSentForReview = true` parks the release

- **Symptom:** The upload reports success. The release sits in Play Console under changes ready to
  send for review and never ships.
- **Cause:** `edits.commit(changesNotSentForReview = true)` writes the track and skips the review
  hand-off. It succeeds on apps that do send changes to review, so scripts that try it first never
  reach their plain-commit fallback.
- **Fix:** Commit without the flag. Use it only when the API demands it, and tell a human to press
  "Send changes for review". Then read the track back with a read-only account; the proof is
  `status: completed` with your versionCode. The first production release of a never-published
  app cannot be confirmed through the API; send it from Play Console.

```python
svc.edits().tracks().update(packageName=pkg, editId=edit, track="production", body={
    "releases": [{"versionCodes": [str(code)], "status": "completed"}]}).execute()
svc.edits().commit(packageName=pkg, editId=edit).execute()   # no changesNotSentForReview
```

- **Seen on:** Play Developer API v3 (androidpublisher).

## 5. If you upload Play listing images through the API: order, inheritance, aspect ratio

- **Symptom:** Screenshots show in the wrong order, a locale shows another locale's images, the
  icon upload fails, App Store frames do not fit Play's limits.
- **Cause:** Upload order is display order. Locales without their own images inherit the default
  locale. The 512 px icon must have an alpha channel. Google Play documents a maximum screenshot
  aspect ratio of 2:1 (the long side at most twice the short side); App Store phone frames such as
  1290x2796 are 2.17:1.
- **Fix:** `images.deleteall`, then upload in order. Crop App Store frames to 2:1 by trimming
  empty background, never by scaling (1290x2796 to 1290x2580). Convert the icon to 32-bit RGBA.
  Verify each upload against the `sha1` that `images.list` returns. The 1024x500 feature graphic
  has no iOS equivalent.
- **Seen on:** Play Developer API v3.

## 6. If you restrict countries: App Store Connect API yes, Play API no

- **Symptom:** A script that sets Play countries fails, or changes nothing.
- **Cause:** In the Play API, `edits.countryavailability` is read-only, and `countryTargeting` on
  the production track fails with "Country targeting is only supported for staged releases".
  The App Store Connect API edits availability per territory and patches only the ones you list;
  new territories still opt in automatically.
- **Fix:** Set Play countries in Console (Production, Countries/regions), then save and send for
  review. Script the iOS side. The two country lists differ, so check both.
- **Seen on:** Play Developer API v3, App Store Connect API.

## 7. If a server or script calls a store API: one service account per job, least privilege

- **Symptom:** A backend that only reads the live version holds a key that can publish releases.
- **Cause:** Reusing the upload service account is the easy path.
- **Fix:** Create a separate account for each job (`gcloud iam service-accounts create`, keys with
  `gcloud`), invite it in Play Console with "View app information" only, and keep the upload key on
  the release machine. `edits.insert` without a commit changes nothing live, so a read-only
  account can inspect tracks and listings. Apple's public lookup endpoint needs no key.
- **Seen on:** Play Developer API v3, Play Console user permissions.

## 8. Play App Signing changes the certificate your installs carry

- **Symptom:** App Links open the browser for store installs but work for sideloads. Attestation
  tokens are missing for every release build.
- **Cause:** With Play App Signing, a Play install is signed by Google's app signing key, not your
  upload key. App Links verify against the installed certificate. Play Integrity attests only
  Play-recognised installs, so a sideloaded release build gets no valid token.
- **Fix:** Put the app signing SHA-256 (from Play Console) in `assetlinks.json` and in Firebase
  next to the upload and debug ones. If you use app attestation, run the server check in monitor
  mode (verify and log, never reject) until store-delivered builds (Play track, TestFlight) show valid tokens, then enforce.
  Enforcing earlier rejects every request, App Review included. Exempt health checks, store
  webhooks and the update check. Release entitlements: see kmp-ios-build.
- **Seen on:** Firebase App Check with Play Integrity and App Attest, Android App Links.

## 9. If you sell in-app purchases on iOS, submit them with the build; a submitted version is locked

- **Symptom:** Rejection under guideline 2.1(b). A submission refuses new items. Screenshot set
  creation fails.
- **Cause:** The first in-app purchases or subscriptions must be reviewed in the same submission
  as a build that sells them. A submission with unresolved issues accepts no new items. Screenshot
  sets cannot be created while the version sits in a review submission.
- **Fix:** Put the version, each subscription and the subscription group in one submission. After
  a rejection, cancel the old submission, create a new one, add every item, submit. To change
  screenshots, remove the version item, upload, and add it back. Upload-time checks
  (launch screen, purpose strings): see kmp-ios-build.
- **Seen on:** App Store Connect API review submissions.

## 10. If your app uses generative AI: the age rating has no AI field; guideline 5.1.2(i) applies

- **Symptom:** You look for an "AI chatbot" question in the age rating and cannot find it.
- **Cause:** The declaration's capability fields are `messagingAndChat`, `userGeneratedContent`,
  `socialMedia`, `advertising`, `parentalControls`, `ageAssurance`, `unrestrictedWebAccess`,
  `lootBox`, `gambling`, `healthOrWellnessTopics` and content severities. None is about AI.
- **Fix:** `messagingAndChat` means user-to-user. With no user-to-user channel, `false` is
  defensible; say in the review notes that the chat partner is an AI. For 5.1.2(i), tell users in
  the app that personal data goes to a third-party AI service, and get consent before the app
  first sends any user data to that service. Name companies, not model versions.
- **Seen on:** App Store Connect age rating declaration, read with the App Store Connect CLI `asc`
  4.4.3.

## 11. If you force updates, decide on the server, per platform

- **Symptom:** A store outage blocks app starts. A typo in the minimum version locks out every
  install. A release name that is a date sorts above every real version.
- **Cause:** The iTunes lookup endpoint is CDN cached, rate limited and has no build number. Play
  release names are free text; only `versionCode` orders builds. A staged rollout is `inProgress`,
  not live everywhere.
- **Fix:**
  - Poll both stores on a server timer into one row per platform; the app asks your server.
    Add a cache-busting query parameter to the lookup call.
  - Android: compare `versionCode` from `completed` releases only. iOS: compare marketing versions.
  - Per platform: prompt `off` (default) or `soft`, plus an explicit minimum version. Enforce the
    minimum only after the store is seen serving it.
  - Client: check on launch and on foreground, fail open, and make a required sheet impossible to
    dismiss. Keep the route free of auth and attestation.
- **Seen on:** iTunes Search/Lookup API, Play Developer API v3.

## Checklist

- [ ] Debug `applicationIdSuffix` chosen on purpose; shortcuts per variant; billing tested from a testing-track install
- [ ] `preReleaseBuild` fails on a blank release secret; debug secrets pinned empty in release
- [ ] versionCode bumped; iOS build number from Apple's highest across all versions
- [ ] iOS marketing version above every approved one
- [ ] Play commit without `changesNotSentForReview`; track read back as `completed`
- [ ] Play countries set in Console; iOS territories set by API
- [ ] Server store access through a read-only account; upload key stays off servers
- [ ] App signing SHA-256 in `assetlinks.json` and Firebase
- [ ] If you use app attestation, it is monitored before it is enforced
- [ ] First subscriptions in the same submission as the build
- [ ] If generative AI: disclosure and consent before user data first goes to the AI service
- [ ] Force update per platform, off by default, minimum enforced only when the store serves it
- [ ] Per-store product ids: see kmp-revenuecat.
