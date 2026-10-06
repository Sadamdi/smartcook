# AGENTS.md — SmartCook

Release rules and the auto-update contract. Read this before shipping any APK.

## Repo layout

`smartcook/` is a superproject with two git submodules, both on `main`:

| Path | What it is |
| --- | --- |
| `smartcook-frontend` | Flutter app (Android). Public: `ChillGuyAdit/smartcook-frontend` |
| `smartcook-backend` | Node/Express API + release manifest. Runs on the VPS under PM2 as `smartcook-backend` |

VPS: `root@192.154.111.198 -p 22232`. There is **no working OpenSSH on this
machine** — use the Paramiko helpers in `%LOCALAPPDATA%\Temp\sc-ssh\`
(`sc_ssh.py` for commands, `sc_put.py` to upload). Do not try plain `ssh`.

Public API: `https://api.himatif-encoder.com` (Cloudflare named tunnel, permanent).

---

## The one rule that breaks updates if you ignore it

**The Android `versionCode` MUST equal the `+N` in `pubspec.yaml`.**

`1.0.10+11` → `versionCode` must be `11`.

Reason: the app reports `versionCode` to `GET /api/app/version`, and the server
publishes the pubspec build in `latest.json`. If those numbers are from
different numbering schemes, the comparison `installedBuild < minBuild` can
never be true, `mandatory` is permanently `false`, and the update dialog never
appears for anyone. This actually shipped broken in v1.0.4 – v1.0.8.

So: **never** build release APKs with `--split-per-abi`. That flag makes the
Flutter Gradle plugin rewrite `versionCode` to `abiVersionCode * 1000 + build`
(build 11 becomes 2011 on arm64, 1011 on arm32).

Build one APK per `--target-platform` instead. `scripts/build_android_release.ps1`
does this and asserts `versionCode == pubspec build` on every build, so a drift
fails the build instead of shipping.

The release script exits non-zero if an ABI build fails. Never work around it —
a stale `app-release.apk` from a previous run will still be sitting in
`build/app/outputs/flutter-apk/` and would silently get published.

---

## Releasing a new version

1. Bump the version in `pubspec.yaml` (`version: X.Y.Z+N`, `N` strictly
   increasing). Never reuse a build number — Android refuses to install a
   lower `versionCode` over an existing one.

2. Build both ABIs:

   ```powershell
   cd smartcook-frontend
   powershell -NoProfile -ExecutionPolicy Bypass -File .\scripts\build_android_release.ps1
   ```

   Output: `build/releases/smartcook-X.Y.Z+N-arm64.apk` and `-arm32.apk`.
   Confirm the script printed `OK versionCode=N` twice.

3. Commit and push the code, then upload the APKs to the server:

   ```powershell
   cd smartcook-frontend
   git add -A && git commit -m "[chore] release X.Y.Z+N"
   git push origin main
   ```

4. Publish `/root/smartcook-releases/latest.json` on the VPS. Keep every field
   consistent — `build`, `minBuild`, and each `sha256` must match reality:

   ```json
   {
     "version": "1.0.10",
     "build": 11,
     "minBuild": 11,
     "blockedBuilds": [],
     "releaseType": "patch",
     "date": "2026-10-07",
     "notes": "One-paragraph summary shown in the update dialog.",
     "apks": [
       { "abi": "arm64", "file": "smartcook-1.0.10-arm64.apk", "sha256": "<sha256>", "sizeBytes": 0 },
       { "abi": "arm32", "file": "smartcook-1.0.10-arm32.apk", "sha256": "<sha256>", "sizeBytes": 0 }
     ],
     "history": [ { "version": "1.0.10", "build": 11, "date": "...", "type": "patch", "notes": "..." } ]
   }
   ```

   `history` is what the in-app changelog sheet renders, newest first. Add an
   entry for every release, including patch releases.

   `minBuild` is the floor below which the update is **mandatory** (dialog
   cannot be dismissed). Set it equal to `build` for a normal release. Raise it
   above `build` only for a security fix that must land immediately.

5. Copy the APKs into `smartcook-frontend/releases/` so they are downloadable
   from GitHub, and create the GitHub Release. `gh` is authenticated as
   `Sadamdi`.

   ```powershell
   Copy-Item build\releases\smartcook-X.Y.Z+N-arm64.apk releases\
   Copy-Item build\releases\smartcook-X.Y.Z+N-arm32.apk releases\
   git add releases\ && git commit -m "[chore] release X.Y.Z+N"
   git push origin main
   gh release create vX.Y.Z --title "SmartCook Android X.Y.Z" `
     --notes-file <file> `
     "releases/smartcook-X.Y.Z+N-arm64.apk" "releases/smartcook-X.Y.Z+N-arm32.apk"
   ```

6. Finally, commit the submodule pointers in the superproject so the parent
   repo references the right commits.

---

## Verifying a release

Never assume. Check these:

```bash
# every build below minBuild must be mandatory
for b in <old build> <previous build>; do
  curl -s -H "X-Smartcook-Cert: $CERT" -H "X-Smartcook-Build: $b" \
    http://127.0.0.1:2122/api/app/version
done
# expect mandatory=true for anything older than minBuild,
#        mandatory=false for the current build.
```

`CERT` is the release signing certificate SHA-256 in `.env` as
`APP_RELEASE_CERT_SHA256`. Without it the server returns no download token, so
an unofficial build cannot update.

Also confirm the manifest hashes match the files actually on disk, and that the
published GitHub asset sizes equal the local file sizes.

---

## API security model

- No static API key in the app. Removed in v1.0.4.
- On launch the app calls `POST /api/auth/handshake`, gated on the
  `X-Smartcook-Cert` header proving the APK carries the release signature. It
  receives an access token (24h) and a refresh token (7d), stored in
  `flutter_secure_storage`.
- Requests send `Authorization: Bearer <access>` and `X-User-Token: <jwt>`.
  `ApiService` refreshes automatically on `TOKEN_EXPIRED`.
- Token keys rotate **manually** via `scripts/rotate-app-tokens.js` on the
  server. Never rotate automatically on restart — that would invalidate every
  user's session on every deploy.
- Legacy `x-api-key` acceptance ends `2026-11-05T03:36:47Z`
  (`APP_LEGACY_KEY_UNTIL`). Do not extend it silently.

---

## Gotchas learned the hard way

- **`--split-per-abi` breaks the update check.** Covered above. It is the single
  most important thing in this file.
- **The API wraps payloads as `{success, data}`.** Read fields off the `data`
  object. Reading off the response root silently yields `null` for everything,
  which looks exactly like "already up to date".
- **`package_info_plus` on Android returns `versionCode`,** not the pubspec
  build. Fine now that `versionCode == pubspec build`, but it is why the
  `--split-per-abi` build broke.
- **`git status` may show many modified files on Windows** when only line
  endings changed. Confirm with `git diff --ignore-cr-at-eol --stat` before
  concluding there are real changes; `git add --renormalize` clears them.
- **The emulator is x86_64.** Release APKs are arm only, so a downloaded
  update will not install there. That is an environment limitation, not an app
  bug. Test dialog behaviour on a real phone.
- **Play Protect blocks sideloaded updates on the emulator.** If installing the
  downloaded APK fails with `INSTALL_FAILED_VERIFICATION_FAILURE`, that is the
  emulator, not the release.

## Localisation

`lib/core/l10n/strings.dart` holds a `Str` interface with hand-written `StrId`
and `StrEn` implementations. Every user-visible string belongs there — including
the update dialog, which was hardcoded Indonesian until v1.0.10. Adding a
language means adding a class and registering it in `stringsFor()`.