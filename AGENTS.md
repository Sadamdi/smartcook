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

All releases go through **one** script: `smartcook-frontend/scripts/release.ps1`
(full guide: `smartcook-frontend/docs/RELEASING.md`). Do not hand-edit
`latest.json` on the VPS and do not run `build_release_manifest.py` there.

```powershell
cd smartcook-frontend
.\scripts\release.ps1 -Type patch      # patch | minor | big | major
```

1. First run creates `release-notes/<build>-<version>.md` and stops. Fill it in
   (rules below), run the same command again.
2. The script bumps `pubspec.yaml` (`+N` rises by exactly 1), builds arm64 +
   arm32 via `build_android_release.ps1` (asserts `versionCode == +N`), checks
   the release signature, generates `latest.json` (it refuses if the newest note
   is not the pubspec build), uploads atomically, keeps only the current and
   previous APKs on the server, then commits, tags `app-vX.Y.Z+N` and pushes.
3. Commit the submodule pointers in the superproject.
4. Run the checks in "Verifying a release". A release is **not done** until they pass.

**One number.** `pubspec +N` = Android `versionCode` = `latest.json` `build` =
the number in the note's file name. There is no second numbering scheme (v1.0.0 -
v1.0.13 shipped with manifest builds 17-28 against versionCodes 1-14, so every
phone was "behind" and forced to update; fixed 2026-10-08). `minBuild` is
computed from notes with `mandatory: true`; `blocks: [..]` fills `blockedBuilds`.

Other modes: `-PublishOnly` (re-publish current build), `-Rollback -To X.Y.Z
-Block N` (roll forward from tag `app-vX.Y.Z+*`; Android cannot downgrade),
`-Unpublish` (restore previous `latest.json`).

### Definition of done for anything user-facing

- Released through `release.ps1` (never a manual upload).
- Release note written and committed (`release-notes/` is tracked; never ignore it).
- Only app changes appear in notes. Server/backend changes (API, database,
  push, email) go in the backend's own docs, and a backend-only change needs
  no APK.
- All touched repos committed and pushed to `main`, tag pushed, submodule
  pointers committed.
- Server verified (next section).

### Standing release approval (owner, 2026-10-08)

The owner has **pre-approved releasing** after a change is rechecked. An agent
that finishes a user-facing change does not need to ask "shall I release?" -
it runs the checklist below and, if every item passes, releases (APK via
`release.ps1`, then verifies). The agent reports what it released afterwards.

Checklist, in this order. Stop and tell the owner if any item fails:

1. **Recheck.** Re-read your own diff and look for the same class of bug next
   to what you changed (a fix is not done until its neighbours were looked at).
2. **Test.** Frontend: `flutter analyze lib` has 0 errors and 0 warnings, and
   `flutter test` passes if tests exist. Backend: `node --check` on touched
   files plus a unit/behaviour test for new logic (never run anything that
   writes to the production database; see Data safety).
3. **Commit and push** every touched repo to `main`; the working tree is clean.
4. **Release note** written and committed (rules in "Release notes"): app
   changes only, Indonesian, no internal jargon.
5. **Release** with `release.ps1 -Type patch|minor` (see above). Create the
   GitHub Release with both APKs.
6. **Verify** on the VPS ("Verifying a release"): current build is not
   mandatory, the previous build gets the optional update, hashes and sizes
   match, `/api/health` is OK. Then commit the submodule pointers.

Still ask first (this approval does **not** cover them): a `major` release, a
`mandatory: true` / raised `minBuild`, a rollback, any database migration or
bulk delete, rotating tokens or keys, anything that spends money or messages
real users, and changes that need a secret only the owner has.

Backend-only changes need no APK: push to `main` and the auto-deploy ships
them after the same recheck/test steps.

### Emulator hygiene

The Android emulator eats RAM. Start it only for the check that needs it and
**shut it down (`adb emu kill`) as soon as that work is truly finished**, then
confirm no `qemu-system*`/`emulator*` process is left. Release APKs are arm
only; for an emulator build use `flutter build apk --release --target-platform
android-x64` (release-signed, so the handshake is accepted) and never publish it.

### Data safety

- Read a script before running it (`deleteMany`, `DROP`, `TRUNCATE`, `rm -rf`).
- **The laptop `.env` is a copy of the server's, so it points at PRODUCTION
  MongoDB.** Anything you run locally that writes (`npm run seed`, a test, a
  one-off script) writes to production. `src/seed.js` now refuses any
  non-local `MONGODB_URI` unless `SEED_CONFIRM_WIPE=<host>` is set, and it
  deletes every recipe and ingredient: back up first. Never run anything that
  bulk-deletes or resets data against production without an explicit "yes"
  from the owner for that one command.
- No scheduled MongoDB backup exists yet; take one (`mongodump`) before any
  bulk write.

### Secrets and `.env`

- `.env` is **never committed** (gitignored) and **never deployed by git**: the
  auto-deploy leaves it alone.
- The owner wants the laptop `.env` identical to the server's, as a backup for
  moving servers. The server is the source of truth. After any change on the
  server run `smartcook-backend\scripts\pull-env.ps1`; it keeps the previous
  file as `.env.pre-sync.local` and never pushes anything to the server.
- Because the laptop now holds production secrets (including
  `DEVLOG_PRIVATE_KEY`), keep the disk encrypted (BitLocker) and never paste
  `.env` contents into chat, tickets or commits.
- Also back up, outside git: the release keystore + `android/key.properties`
  (losing the keystore means no existing install can ever be updated) and the
  Firebase service-account json.

## Encrypted API channel (all endpoints)

HTTPS ends at Cloudflare, which can read every request. The app therefore seals
**every API request and its answer** to the server's public key (X25519 +
HKDF-SHA256 + AES-256-GCM, ephemeral key per request). Cloudflare only sees
`POST /api/secure` with random bytes. Code: `smartcook-backend/src/modules/secure/channel.js`
(server) and `smartcook-frontend/lib/core/services/secure_channel.dart`
(`SecureHttpClient` for `ApiService`/chat, `SecureDioInterceptor` for the
session endpoints). Change both sides together.

- The private key is `API_PRIVATE_KEY` in the server `.env` (and, as a backup
  copy, the laptop `.env`). The app holds only the public key + `kid`
  (`API_PUBLIC_KEY` / `API_KEY_ID` dart-defines override the defaults).
- Stays plaintext on purpose: `GET /api/app/*` (update check/download, so an old
  or blocked app can always see the update dialog), `GET /api/health`,
  `POST /api/devlog/ingest` (own envelope). Same list on both sides.
- Provides confidentiality, integrity and replay protection (5 min window +
  nonce cache). It does **not** authenticate the client; the public key is in
  the APK. The handshake/session gates still apply inside the channel.
- **Rollout.** The server accepts plaintext AND sealed requests until
  `API_REQUIRE_ENCRYPTED=1`. Old apps keep working. To make encryption
  mandatory: (1) check adoption (live sessions by build, see below), (2) ship a
  release with `mandatory: true` / raised `minBuild` for the first sealed build
  (this needs the owner's explicit OK, see Standing release approval),
  (3) set `API_REQUIRE_ENCRYPTED=1` in the server `.env` and restart PM2. After
  that every plaintext call except the exempt ones answers 426
  `UPDATE_REQUIRED`, whose message the old app shows to the user.
- **Key rotation.** Generate a new pair on the server, put the NEW private key in
  `API_PRIVATE_KEY` and the old one in `API_PRIVATE_KEY_PREV`, ship an app with
  the new public key/`kid`, and drop `_PREV` only when old builds have aged out.
- A phone clock off by more than 5 minutes is handled: the client learns the
  offset from the server and retries once.
- `SECURE_API=false` (dart-define) turns sealing off for development against a
  server without the key. Never ship that.
- Tests (run all before touching it): backend `node scripts/test-secure-channel.js`;
  frontend `flutter test` (unit) plus the cross-language interop run, which
  starts the real Node stack: `scripts/secure-interop-server.js` with
  `INTEROP_API_PRIVATE_KEY`, then `flutter test test/secure_channel_interop_test.dart`
  with `SECURE_INTEROP_URL`, `SECURE_INTEROP_STRICT_URL`, `SECURE_INTEROP_PUB`,
  `SECURE_INTEROP_KID` set.
- Adoption check (read-only), run on the VPS: count live app sessions by build,
  e.g. in `/root/smartcook-backend` query the `apptokens` collection grouped by
  `build`. Builds below the first sealed build are still plaintext.

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
- **`adb install` can report Success without updating the app.** Check
  `dumpsys package com.example.smartcook | grep lastUpdateTime` before trusting
  a test result. Uninstall first when the signature changes.
- **uiautomator cannot read Flutter text.** A Flutter screen with many widgets
  dumps zero text nodes; that is not proof of a blank screen. Sample screenshot
  pixels instead (`distinct colours` > 1 means it rendered).
- **Dio throws on non-2xx before the body is readable.** Any check that needs
  the error body (`code` like `FORBIDDEN_CLIENT`) must set
  `validateStatus: (s) => s < 500`, otherwise it is misread as a network fault.
- **Never change `locale` while a dialog is open.** It rebuilds MaterialApp and
  tears the dialog down mid-`Navigator.pop`. Pop first, switch on the next frame.
- **Any await on the splash path needs a failure route.** It used to have none,
  so a single error left the user on a white screen with no way forward.

## Developer debug log (telemetry)

Answers "which phone, which build, which action, what error" for a bug report
without asking the user to run anything. **Not a user-facing feature — keep it
out of release notes.**

Client: `smartcook-frontend/lib/core/services/dev_log.dart`, posting batches to
`POST /api/devlog/ingest`. Server: `smartcook-backend/src/modules/devlog/`.

Captured per event: app launch/resume, failed API calls (path + status), render
errors, unhandled Flutter errors, session handshake outcome, language switch.
Each row carries device model, manufacturer, OS version, SDK level, ABI, device
locale, app version, build, plus the server's view — real client IP, user
agent, signing certificate. With a valid `X-User-Token` the account is attached.

Queries are made from the server shell, e.g.:

```bash
cd /root/smartcook-backend && node -e "
require('dotenv').config();
const mongoose=require('mongoose');
const DevLog=require('./src/modules/devlog/model');
(async()=>{ await mongoose.connect(process.env.MONGODB_URI);
  const rows=await DevLog.find({level:'error'}).sort({createdAt:-1}).limit(20).lean();
  for(const r of rows) console.log(r.createdAt.toISOString(), r.event, r.action,
    '|', r.deviceManufacturer, r.deviceModel, 'os'+r.osVersion, 'b'+r.appBuild,
    '|', (r.error||'').slice(0,80));
  await mongoose.disconnect(); })();
"
```

Useful filters: `event`, `level`, `installId`, `userId`, `appBuild`, `createdAt`.

Rules that must stay true:

- **Retention is 30 days** (`DEVLOG_RETENTION_DAYS`), enforced by a MongoDB TTL
  index on `expiresAt`. `expiresAt` is `required`, not defaulted: a TTL index
  silently ignores documents missing the field, so a default would keep rows
  forever.
- **Identity is never read from the payload.** It comes from the verified token,
  so a modified client cannot log events as another user.
- `/api/devlog/ingest` must stay in `OPEN_PATHS` in `server.js`. A crash during
  boot happens *before* the handshake, and those are the events worth having.
- Event names are allow-listed and every string is length-capped.
- **Batches are encrypted** (X25519 + HKDF-SHA256 + AES-256-GCM sealed box,
  `smartcook-backend/src/modules/devlog/crypto.js`). The app holds only the
  server PUBLIC key; the private key is `DEVLOG_PRIVATE_KEY` in the server
  `.env`. Each batch has its own ephemeral key; `kid` allows rotation (put the
  old key in `DEVLOG_PRIVATE_KEY_PREV` until old APKs age out). Plaintext
  batches from old APKs are accepted until `DEVLOG_REQUIRE_ENCRYPTED=1`.
  Encryption gives confidentiality and integrity, not authenticity: a modified
  client can still send bogus (size-capped, allow-listed) events. Run
  `node scripts/test-devlog-crypto.js` and `scripts/test-devlog-ingest.js`
  after touching it.
- **Privacy in logs.** The collection stores only the opaque `userId` (no name,
  no email) and the client IP masked to /24. Raw PM2 logs are redacted by
  `src/utils/redact.js` (emails become `email:<hash>`, IPv4 keeps /24) and
  `pm2-logrotate` rotates daily and keeps 30 files, so nothing outlives 30
  days. Never log request/response bodies, tokens, OTPs or typed text.
- Retention is verified, not assumed: the TTL index on `expiresAt` exists and
  no row lacks `expiresAt`.
- Wire keys are single letters. **If you change one, change `WIRE` in
  `src/modules/devlog/service.js` and `_k*` in `dev_log.dart` together** — a
  mismatch silently drops every event (`accepted:0, rejected:N`).

## Version numbering (`pubspec.yaml:version`)

| Bump | Contoh | Kapan | Mandatory |
| --- | --- | --- | --- |
| PATCH | `1.0.10+11` → `1.0.11+12` | bug fix, teks, performa, keamanan minor | opsional |
| MINOR | `1.0.x` → `1.1.0` | fitur baru kecil/sedang | opsional |
| BIG | `1.x` → `1.5.0` → `2.0.0` | paket fitur besar / redesign | opsional (wajib: gunakan BIG kalau redesign major) |
| MAJOR | `2.0.0+0` → `3.0.0+0` | rewrite, API break | **wajib** |

Build number (`+N`) **selalu naik**. Tidak pernah reuse, tidak pernah turun.
A `pubspec.yaml` dengan `+N` lebih kecil dari rilis yang sudah diterbitkan
ditolak oleh server, dan Android sendiri menolak downgrade tanpa uninstall
manual dulu.

Default `mandatory` di manifest baru: `false`, kecuali `type=major` yang
`true`. Bisa dipaksa ke `true` lewat `minBuild` untuk keamanan darurat.

## Release notes (frontend user-facing)

Catatan rilis user-facing hidup di
`smartcook-frontend/release-notes/<build>-<version>.md`. Bahasa Indonesia,
nada sopan, **tanpa jargon internal**:

- Tidak ada nama developer, commit hash, atau referensi ke file.
- Tidak ada kata "server", "token", "fingerprint", "debug log",
  "telemetry", "fix", "bug", "Crashlytics".
- Bullet ringkas, satu kalimat, tidak semuanya mulai dengan kata yang sama.
- Maksimal 3-4 poin per section (Yang baru, Perbaikan, dst).
- Tidak ada emoji berlebihan; kalau pakai, satu saja.

Format file (frontmatter YAML + body):

```
---
version: 1.0.12
build: 13
date: 2026-10-07
type: minor
mandatory: false
sections:
  - kind: new
    items:
      - Tambah mode dapur dengan timer built-in, tidak perlu lagi pakai HP kedua saat masak.
      - Rekomendasi resep sekarang menghargai diet user: vegetarian, vegan, rendah gula.
  - kind: fix
    items:
      - Login Google tidak lagi gagal saat akun HP baru pertama kali ditautkan.
      - ...
---
Headline satu kalimat tentang rilis ini.

The rest of the body is shown when the user taps "See full notes".
```

Manifest server (`/root/smartcook-releases/latest.json`) menerima field
`headline` + `sections[]` dan menurunkan body field `notes` lama jadi ringkasan,
jadi rilis-rilis lama masih tetap tampil layak sampai keluar dari window
history.

## Server messages and e-mails follow the app language

The app sends `X-Smartcook-Locale` (`id`/`en`) on every API call (it travels
inside the sealed channel too). `smartcook-backend/src/utils/i18n.js` swaps the
`message` of each JSON answer for its English text and `sendOTPEmail` takes
`lang`, so OTP e-mails and errors such as "Kode OTP salah." match the language
the user picked. Controllers keep writing Indonesian; **every new message needs
an entry in `EN` in `i18n.js`** (and both languages in the e-mail templates).
`node scripts/test-i18n.js` fails when a message in `src/` has no English text.

## Backend auto-deploy and pm2

systemd runs `scripts/deploy.sh` without `HOME`; unpinned, `pm2` talks to an
empty daemon and a deploy "succeeds" without restarting (fixed 2026-10-09: the
unit and script pin `PM2_HOME=/root/.pm2`, a failed restart fails the deploy).
After a backend push, confirm the process uptime reset (`pm2 jlist`).

## 16 KB page size (Android 15+)

What Google Play checks passes: every arm64 library has `p_align` >= 16 KB and
the zip passes `zipalign -c -P 16 -v 4 <apk>` (checked 2026-10-09 on 1.1.2).
Flutter 3.44, AGP 8.11, NDK 28 and androidx.datastore 1.2.0 (latest) are already
the 16 KB-ready versions. The runtime dialog "This app isn't 16 KB compatible"
still shows on the 16 KB x86_64 emulator image for the dev build; it names
`libdatastore_shared_counter.so` (AndroidX prebuilt, its RELRO end is not
16 KB aligned and we cannot rebuild it) plus engine/jni libs. A real 16 KB ARM
phone cannot be tested from this machine, so re-check after any Flutter, AGP or
plugin upgrade (`zipalign` + ELF `p_align`/`PT_GNU_RELRO`) and bump androidx
datastore when a newer one ships.

## Localisation

`lib/core/l10n/strings.dart` holds a `Str` interface with hand-written `StrId`
and `StrEn` implementations. Every user-visible string belongs there — including
the update dialog, which was hardcoded Indonesian until v1.0.10. Adding a
language means adding a class and registering it in `stringsFor()`.