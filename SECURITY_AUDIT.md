# SmartCook — End-to-End Security & Feature Audit

Scope: read-only audit of `smartcook-backend/` (Node + Express + Mongoose +
Gemini) and `smartcook-frontend/` (Flutter). Build only from the source tree and
`git log`. No code, APK, or VPS state was changed.

## A. Backend hardening checklist

| # | Item | Status | Location |
|---|------|--------|----------|
| 1 | `helmet()` mounted | present | `smartcook-backend/server.js:50` |
| 2 | Global `/api/` rate limiter (1000/min/IP) | present | `smartcook-backend/server.js:73-106` (mounted at line 109) |
| 3 | Dedicated `/api/chat` rate limiter (20/min/IP) | present | `smartcook-backend/server.js:110-146` (mounted at line 147) |
| 4 | Dedicated `/api/auth/google` rate limiter (30/hour/IP) | present | `smartcook-backend/src/routes/auth.js:19-30` (mounted at line 33) |
| 5 | Dedicated `/api/app/version` rate limiter (120/hour/IP) | present | `smartcook-backend/src/modules/app/routes.js:11-22` (mounted at line 24) |
| 6 | JWT secret length check (>=32) | not enforced at startup; `process.env.JWT_SECRET` is used directly. Length is only an operational convention, not a runtime guard | `smartcook-backend/src/controllers/authController.js:9`, `smartcook-backend/src/lib/googleTicket.js:29-34` |
| 7 | `bcryptjs` password hashing + compare | present | `smartcook-backend/src/models/User.js:2, 110-114, 116-119` |
| 8 | Login OTP rate limit (5x/5min, 10/day, per-user + per-IP) | present and wired into `POST /api/auth/login` | `smartcook-backend/src/controllers/authController.js:22-26, 39-100, 136-200, 410-498` |
| 9 | Response shape `{ success, message, code }` and stack hidden when `NODE_ENV === 'production'` | present; stack only added when env is `'development'` | `smartcook-backend/src/middleware/errorHandler.js:23-28` |
| 10 | CORS is permissive (`cors()` no allowlist) | confirmed — flag as risk | `smartcook-backend/server.js:51` |
| 11 | `helmet()` called with default config (no `contentSecurityPolicy` set) | confirmed — flag as risk | `smartcook-backend/server.js:50` |
| 12 | JSON body size limit `10mb` | present | `smartcook-backend/server.js:52` |
| 13 | `/api/auth/*` does NOT require `x-api-key`; it uses `Authorization: Bearer <jwt>` | confirmed; `validateApiKey` short-circuits on `req.path.startsWith('/api/app/')` and the `/api/auth` router is mounted directly without an additional API-key middleware. Bearer-token protection is per-route via `protect` on `/api/auth/google/set-password` | `smartcook-backend/server.js:55-62`, `smartcook-backend/src/middleware/apiKey.js:1-23`, `smartcook-backend/src/routes/auth.js:35` |
| 14 | `/api/app/*` requires `X-Smartcook-Cert` matching `APP_RELEASE_CERT_SHA256`, else 403 | confirmed via cert compare + `FORBIDDEN_CLIENT` 403 | `smartcook-backend/src/modules/app/service.js:36-51, 103-138`; flow `smartcook-backend/src/modules/app/controller.js:6-23` |

## B. Frontend checklist

| # | Item | Status | Location |
|---|------|--------|----------|
| 1 | JWT + user cached in `SharedPreferences` via `TokenService` (plaintext — flag for production hardening) | confirmed | `smartcook-frontend/lib/service/token_service.dart:1-39` |
| 2 | `api_config.dart` ships `baseUrl` + `x-api-key`; `_headers({useAuth:true})` adds `Authorization: Bearer <jwt>` | confirmed | `smartcook-frontend/lib/config/api_config.dart:1-13`, `smartcook-frontend/lib/service/api_service.dart:18-28` |
| 3 | `dio` timeouts `connectTimeout: 15s`, `receiveTimeout: 20s` | confirmed | `smartcook-frontend/lib/core/services/app_update_fetcher.dart:79-81` |
| 4 | TLS verification enabled by default (no `badCertificateCallback`, no `HttpClient` disable) | confirmed by absence — no `HttpOverrides` override, `http` package uses Dart's defaults | n/a |
| 5 | Android `network_security_config.xml` | not present — recommend adding it to enforce HTTPS-only and to declare `cleartextTrafficPermitted="false"` for the auto-update host | `smartcook-frontend/android/app/src/main/AndroidManifest.xml:1-44` (no `<application android:networkSecurityConfig=…/>` line) |

Notes on token storage: storing JWT in `SharedPreferences` is acceptable in this
scaffold, but on Android production should use `flutter_secure_storage`
(Android Keystore + iOS Keychain). See finding F-1.

## C. Backend endpoint inventory

| Method | Path | Auth | Rate limit | Purpose |
|--------|------|------|------------|---------|
| GET | `/api/health` | none | global `/api/` (1000/min/IP) | health probe |
| POST | `/api/auth/register` | none | global | create email account, return JWT |
| POST | `/api/auth/login` | none | global + per-user/per-IP login OTP-limits (5/5min, 10/day) | email+password login |
| POST | `/api/auth/google` | none | dedicated `googleLimiter` (30/hour/IP) | verify Firebase/Google ID token, sign in / sign up |
| POST | `/api/auth/google/set-password` | JWT (`protect`) | global | attach password to a Google account |
| POST | `/api/auth/forgot-password` | none | global + OTP-send rate (60s) | send OTP for reset |
| POST | `/api/auth/verify-otp` | none | global | verify forgot-password OTP |
| POST | `/api/auth/reset-password` | none | global | set new password after OTP |
| POST | `/api/auth/login-otp-verify` | none | global | verify login OTP after daily limit, optional reset |
| POST | `/api/auth/login-otp-resend` | none | global + OTP-send rate (60s) | resend login OTP |
| GET | `/api/user/profile` | JWT | global | read own profile |
| PUT | `/api/user/profile` | JWT | global | update name/age/gender |
| PUT | `/api/user/onboarding` | JWT | global | save allergy/medical/cooking/equipment prefs |
| POST | `/api/user/password/send-otp` | JWT | global | email OTP for password change |
| POST | `/api/user/password/change` | JWT | global | change password (current or OTP) |
| POST | `/api/user/email/send-otp` | JWT | global | email OTP for email change |
| POST | `/api/user/email/confirm` | JWT | global | confirm new email with OTP |
| GET | `/api/recipes` | none | global | list recipes (paginated, filter by category/tags) |
| GET | `/api/recipes/search` | none | global | text-search recipe |
| GET | `/api/recipes/:id` | none | global | get one recipe, bump popularity |
| GET | `/api/recipes/with-fridge/:id` | JWT | global | recipe + ingredients flagged with availability |
| GET | `/api/recipes/by-meal/:type` | none | global | filter by breakfast/lunch/dinner |
| GET | `/api/recipes/popular` | none | global | top recipes by `popularity_count` |
| GET | `/api/recipes/recommendations` | JWT | global | allergy-aware random recommendations |
| GET | `/api/recipes/ai-search` | JWT | global | AI-generated recipe via Gemini |
| GET | `/api/recipes/query` | JWT | global | existing-only search |
| GET | `/api/recipes/global-search` | JWT | global | cross-user search, allergy-aware |
| GET | `/api/fridge` | JWT | global | list user's fridge items |
| POST | `/api/fridge` | JWT | global | add / increment fridge item |
| PUT | `/api/fridge/:id` | JWT | global | update quantity/unit/expired_date |
| DELETE | `/api/fridge/:id` | JWT | global | delete one fridge item |
| GET | `/api/fridge/by-category/:category` | JWT | global | list by protein/karbo/sayur/bumbu |
| POST | `/api/fridge/bulk-from-recipe/:id` | JWT | global | add missing recipe ingredients to fridge |
| GET | `/api/favorites` | JWT | global | list favorite recipes (paginated) |
| POST | `/api/favorites/:recipeId` | JWT | global | add favorite |
| DELETE | `/api/favorites/:recipeId` | JWT | global | remove favorite |
| GET | `/api/categories/cooking-styles` | none | global | static list |
| GET | `/api/categories/meal-types` | none | global | static list |
| GET | `/api/categories/ingredients` | none | global | ingredient catalog (grouped) |
| GET | `/api/ingredients` | none | global | list/search global ingredient catalog |
| POST | `/api/ingredients` | JWT | global | upsert a global ingredient |
| POST | `/api/chat/message` | JWT | dedicated `chatLimiter` (20/min/IP) | Gemini chat, non-streaming |
| POST | `/api/chat/message-stream` | JWT | `chatLimiter` (20/min/IP) | Gemini chat, SSE stream + recipe embeds |
| GET | `/api/chat/history` | JWT | global | read user's chat history |
| DELETE | `/api/chat/history` | JWT | global | wipe user's chat history |
| GET | `/api/app/version` | none, but API-key middleware skips `/api/app/*` (see `server.js:55-62`); gated by cert | dedicated `versionLimiter` (120/hour/IP) | publish latest version + HMAC download token when cert matches |
| GET | `/api/app/download` | cert gate via HMAC token (`?t=…`) | global | stream per-ABI APK with ETag + Range support |

Note on `/api/recipes/with-fridge/:id`: `protect` requires a valid JWT, but the
route also returns user's `FridgeItem` flags, which is intentionally a
JWT-gated endpoint.

## D. Flutter screens and their backend usage

| Screen | Backend endpoints hit | Error surfacing |
|--------|-----------------------|-----------------|
| `auth/signin.dart` | `POST /api/auth/login`, `POST /api/auth/google` | error text under the form, shake animation; `LOGIN_OTP_REQUIRED` routes to `LoginOtpPage`; `LOGIN_RATE_LIMIT` shows countdown on button. |
| `auth/signUp.dart` | `POST /api/auth/register`, `POST /api/auth/google` | `SnackBar` with `res.message`; for Google flow, branch to `GoogleSetPasswordPage` if `needs_password`. |
| `auth/forgotpassword.dart` | `POST /api/auth/forgot-password` | `SnackBar`; on success navigates to `mailpassword`. Cooldown uses `retry_after_seconds` or 60s default. |
| `auth/mailpassoword.dart` | `POST /api/auth/verify-otp`, `POST /api/auth/forgot-password` (resend) | `SnackBar` on failure, navigates to `resetpassword` on success. |
| `auth/resetpassword.dart` | `POST /api/auth/reset-password` | `SnackBar` on failure, animasi success page. |
| `auth/login_otp.dart` | `POST /api/auth/login-otp-verify`, `POST /api/auth/login-otp-resend` | inline `_errorMessage` text + OTP-box focus; cooldown buttons. |
| `auth/google_set_password.dart` | `POST /api/auth/google/set-password` (JWT) | `SnackBar`. |
| `page/homepage.dart` | `GET /api/user/profile`, `GET /api/favorites`, `GET /api/fridge`, `GET /api/recipes/recommendations`, `GET /api/recipes/popular`, `GET /api/recipes/by-meal/{breakfast,lunch,dinner}` | fallback to `OfflineCacheService` if offline. Offline state set via `OfflineManager.setOffline(...)`. |
| `page/bot_page.dart` | `POST /api/chat/message-stream`, `GET /api/chat/history`, `DELETE /api/chat/history` | SSE parse → `MarkdownBody` + recipe embeds; on HTTP error shows `SnackBar`. Offline page replaces UI entirely. |
| `page/kulkas.dart` | `GET /api/fridge`, `PUT /api/fridge/:id`, `DELETE /api/fridge/:id` | `_showSuccessPopup` on success; `SnackBar` on failure; offline path queues via `OfflineCacheService.addPendingOperation`. |
| `page/tambahkan_bahan.dart` | `GET /api/ingredients`, `POST /api/fridge` (one per item) | snackbars; offline queues each `POST /api/fridge`. |
| `page/category.dart` | `GET /api/recipes?category=…` | local filter; empty-state text. |
| `page/masakan.dart` | `GET /api/recipes/with-fridge/:id`, `GET /api/favorites`, `POST /api/favorites/:id`, `DELETE /api/favorites/:id`, `POST /api/fridge/bulk-from-recipe/:id` | `SnackBar` with `res.message`; offline caches recipe. |
| `page/search_page.dart` | `GET /api/recipes/ai-search`, `GET /api/recipes/query`, `GET /api/recipes/global-search` | debounced, cached via `OfflineCacheService`; empty-state text. |
| `page/save_page.dart` | `GET /api/favorites` | offline fallback to local favorite cache; empty-state component. |
| `page/profile_page.dart` | `GET /api/user/profile`, `PUT /api/user/profile` | snackbars; logout clears token and routes to `/signin`. |
| `page/change_password_page.dart` | `POST /api/user/password/send-otp`, `POST /api/user/password/change` | snackbars; cooldown counters. |
| `page/change_email_page.dart` | `POST /api/user/email/send-otp`, `POST /api/user/email/confirm` | snackbars; cooldown counters. |
| `core/services/app_update_checker.dart` + `core/services/app_update_fetcher.dart` | `GET /api/app/version`, `GET /api/app/download?abi=…&t=…` | structured `UpdateFailure` enum + `UpdateFailureCode`; cert mismatch → `notOfficial` dialog. |

## E. Top 10 ranked findings

| # | Severity | Finding | Location | Suggested fix |
|---|----------|---------|----------|----------------|
| 1 | High | JWT stored in `SharedPreferences` in plaintext on the device. | `smartcook-frontend/lib/service/token_service.dart:7-15` | Move the JWT (and `auth_user` blob) to `flutter_secure_storage` (Android Keystore + iOS Keychain). For Android, also enable EncryptedSharedPreferences. Keep `clearAll()` so logout wipes the secure store. |
| 2 | High | CORS is fully permissive (`cors()` with no allowlist). | `smartcook-backend/server.js:51` | Replace with an explicit allowlist, e.g. `cors({ origin: ['https://api.himatif-encoder.com'], methods: ['GET','POST','PUT','DELETE'], credentials: false })`. The mobile client doesn't need CORS at all (it is not browser-origin), so the allowlist only matters for any future admin web UI. |
| 3 | High | `helmet()` is mounted with default config; no explicit `contentSecurityPolicy`, `crossOriginEmbedderPolicy`, or `referrerPolicy`. | `smartcook-backend/server.js:50` | Pass an options object: `helmet({ contentSecurityPolicy: { directives: { 'default-src': ["'none'"] } }, crossOriginResourcePolicy: { policy: 'same-site' }, referrerPolicy: { policy: 'no-referrer' } })`. Even for a JSON API, the defaults are too lenient for a service that ships APK binaries. |
| 4 | Medium | `/api/chat` limiter is 20/min/IP but the AI endpoint costs real money per call and can be triggered by any valid JWT (see Section F). | `smartcook-backend/server.js:110-146` | Tighten chat limits: 10/min/IP and 60/hour/user, and add a per-user cap (e.g. via Redis or a Mongoose counter on `User`). Also rate-limit `POST /api/recipes/ai-search` with a tighter window — currently it has none. |
| 5 | Medium | `JWT_SECRET` is used by `authController.js` and `googleTicket.js` without a length check at boot; weak secrets silently work. | `authController.js:9`, `lib/googleTicket.js:29-34` | In `server.js` startup, validate `process.env.JWT_SECRET.length >= 32` and fail-fast with `process.exit(1)` if not. Same for `APP_DOWNLOAD_TOKEN_SECRET` and `APP_RELEASE_CERT_SHA256`. |
| 6 | Medium | `x-api-key` is checked with `===` against the env value (timing-leaky on the wrong side, but more importantly it is hard-coded into the APK and shipped in plaintext). | `smartcook-backend/src/middleware/apiKey.js:9-22`, `smartcook-frontend/lib/config/api_config.dart:11` | Treat the key as compromised: rotate it (see Section G) and consider switching to per-installation mTLS or a rotating signed JWT issued by a `/api/handshake` endpoint. At minimum, log every key mismatch and consider HMAC-fingerprint validation. |
| 7 | Medium | `chatController.js` injects user-controlled text (`message`) directly into the Gemini prompt and returns raw model output. There is no input sanitization beyond `message.trim()`. Prompt-injection or PII exfiltration possible if Gemini's instructions drift. | `smartcook-backend/src/controllers/chatController.js:471-540` | Strip obvious PII patterns (emails, phone numbers) before sending; add a system-side reminder that the assistant must refuse to disclose other users' data; cap `message.length` server-side (e.g. 1000 chars) before reaching Gemini; sanitize `fullReply` before logging it. |
| 8 | Medium | Several unauthenticated public endpoints are sensitive: `GET /api/recipes`, `GET /api/recipes/:id`, `GET /api/recipes/popular`, `GET /api/categories/*`, `GET /api/ingredients`, `GET /api/recipes/by-meal/:type`. Anyone with the published `x-api-key` (which is in the APK) can scrape them. | `routes/recipe.js:14-18, 26, 28, 30`, `routes/ingredient.js:9`, `routes/category.js:9-13` | Decide explicitly: either accept that the recipe catalog is public, or move these behind `protect`. At minimum, log per-IP request volume and add `Limiter` keyed on `x-api-key`. |
| 9 | Low | No Android `network_security_config.xml`. The `AndroidManifest.xml` does not set `android:networkSecurityConfig` nor `android:usesCleartextTraffic`. | `smartcook-frontend/android/app/src/main/AndroidManifest.xml:1-44` | Add `android/app/src/main/res/xml/network_security_config.xml` with `cleartextTrafficPermitted="false"` and `<domain-config cleartextTrafficPermitted="false"><domain includeSubdomains="true">api.himatif-encoder.com</domain-config>`. Then add `android:networkSecurityConfig="@xml/network_security_config"` on the `<application>` tag. |
| 10 | Low | Error handler always includes `code` only on the development flag, but several controllers include a precise `code` even on 200/201 (e.g. `OTP_SEND_RATE_LIMIT`). These codes reveal internal state. | `smartcook-backend/src/middleware/errorHandler.js:23-28`, `authController.js:340-345, 450-455` | Reserve structured `code` fields for true client-facing flows (rate-limit / login OTP) and stop echoing `code` for 5xx. For 5xx in production, drop `code` and only log `err.stack` server-side. |

Additional notes (not part of top 10):
- `/api/auth/google` will accept any Google account when `GOOGLE_CLIENT_IDS`
  is empty (see `lib/googleAuth.js:60-73`). This is by design to avoid locking
  out existing users, but it should be tightened when the OAuth web client IDs
  are stable.
- `ipLoginState` in `authController.js:28-35` is an in-memory `Map` — it does
  not survive process restarts and is per-PM2 worker. That is acceptable for
  rate-limiting purposes but worth noting.
- The cert-gate 24h download token in `modules/app/service.js:9-86` uses HMAC
  `sha256`, not signed JWT, so there is no key-rotation grace window. If
  `APP_DOWNLOAD_TOKEN_SECRET` leaks, every outstanding token must be
  invalidated by rotating the secret (see Section G).

## F. Done today (recent commits + deploy)

Backend repo `Sadamdi/smartcook-backend.git`:
- `0a51725` — verify Firebase ID token + GoogleTicket helper.
  Files: `smartcook-backend/src/lib/googleAuth.js`,
  `smartcook-backend/src/lib/googleTicket.js`,
  `smartcook-backend/src/controllers/authController.js`.

Frontend repo `ChillGuyAdit/smartcook-frontend.git` (3 commits):
- `cd4af7f` — add per-ABI APK split in the auto-update manifest and
  `app_update_fetcher.dart` (arm64 / arm32).
- `ee426ab` — Google sign-in fix: pass `serverClientId` to `GoogleSignIn` so
  `idToken` is non-null on Android; bake the OAuth web client ID into
  `lib/core/network/env.dart` via `GOOGLE_WEB_CLIENT_ID`.
- `2742e39` — bake OAuth web client ID + version metadata on profile so the
  APK can identify itself to `/api/app/version`.

VPS deploy (current state, not changed by this audit):
- Backend under PM2 on port 2122.
- Cloudflare named tunnel `smartcook-api` routes `api.himatif-encoder.com` →
  VPS → Node.
- `latest.json` published:
  `version = 1.0.3`, `build = 4`,
  `arm64 sha256 = 1bab03657c558be62f4aff1f17a60e0c4153503eb9fca5bd9f087f809128ef00`,
  `arm32 sha256 = 8a4a0ead30fbe3991b8555375ce6a6f24bff9fb0380d9f7df4b49819ccbf7270`.

AI architecture (Gemini):
- Server-side only. The mobile client never sees `GOOGLE_API_KEY*`.
  Environment vars live on the VPS in `/home/<deploy>/.env` (verified by
  source: `smartcook-backend/src/config/gemini.js:88-103`).
- Mobile reaches the AI through two endpoints, both gated by `x-api-key` +
  JWT (`Bearer`):
  `POST /api/chat/message` and `POST /api/chat/message-stream`. The mobile
  client cannot leak the Gemini key because it never holds it.
- Threat: an attacker who obtains a valid JWT (e.g. by phishing a user) can
  call `/api/chat/message-stream` and trigger Gemini calls at the user's
  expense. The 20/min/IP `chatLimiter` slows but does not stop this — see
  Finding F-4. The right mitigation is a per-user counter, not just per-IP.

## G. Disaster recovery — credential rotation runbook

Read-only: do NOT execute as part of this audit.

1. Rotate `APP_DOWNLOAD_TOKEN_SECRET` (24h HMAC on `/api/app/download`):
   - Generate new secret (`openssl rand -hex 64`) on the VPS.
   - Update `.env` (or PM2 env) with `APP_DOWNLOAD_TOKEN_SECRET=<new>`.
   - `pm2 restart smartcook-api`.
   - Old download tokens issued before rotation will fail with `403
     FORBIDDEN`; users already past the version-check screen will simply see
     a retry prompt. Once the next APK is published, `latest.json` is
     re-signed and clients re-fetch `/api/app/version` to mint a fresh token.
   - If you want to invalidate every in-flight token immediately, also bump
     `latest.json` `build` (publish a no-op patch release with a new
     `version`/`build`) — clients re-fetch and re-issue tokens.

2. Rotate `APP_RELEASE_CERT_SHA256` (cert gate on `/api/app/*`):
   - Re-sign the APK with a new keystore (see step 4).
   - `keytool -list -v -keystore <new.jks> -alias <alias>` → copy the SHA-256
     fingerprint.
   - Update `.env` with `APP_RELEASE_CERT_SHA256=<new-fingerprint>`.
   - `pm2 restart smartcook-api`.
   - Old APKs (with the previous signing cert) will get
     `403 FORBIDDEN_CLIENT` on `/api/app/version` and cannot download
     updates — intentional. Publish a new `latest.json` if you need users to
     upgrade.

3. Rotate `JWT_SECRET`:
   - Generate new secret (`openssl rand -hex 64`).
   - Update `.env` with `JWT_SECRET=<new>`.
   - `pm2 restart smartcook-api`.
   - Effect: all existing JWTs and Google tickets are invalidated. Every
     user is silently logged out and must sign in again. Acceptable as a
     last-resort response to a key leak. Coordinate with on-call before
     rotating during peak hours.

4. Rotate the APK signing keystore (`signing/smartcook-release.jks`):
   - Generate a new keystore (`keytool -genkey -v -keystore new.jks -alias
     smartcook -keyalg RSA -keysize 4096 -validity 25000`).
   - Move the old keystore out of the build path; commit the new keystore
     OUT-of-band (do not push to git).
   - Update `android/key.properties` (or equivalent) with the new alias /
     passwords.
   - Build a new APK and publish it with a bumped `version`/`build` in
     `latest.json`. Note: Android treats a new signing key as a different
     app, so users will need to uninstall + reinstall to upgrade. For
     SmartCook this is acceptable because updates flow through the in-app
     auto-update rather than Google Play.

5. (Bonus) rotate the `x-api-key` shared secret:
   - Generate a new key (`openssl rand -hex 32`).
   - Update backend `.env` `API_KEY=<new>` and `pm2 restart smartcook-api`.
   - Rebuild + re-publish the APK with the new `ApiConfig.apiKey`
     (`smartcook-frontend/lib/config/api_config.dart:11`). Users on the old
     APK will start receiving `403 Akses ditolak. API key tidak valid.` on
     every request until they update.

## Appendix: line counts

| File | LOC |
|------|-----|
| `smartcook-backend/server.js` | 206 |
| `smartcook-backend/src/modules/app/service.js` | 193 |
| `smartcook-backend/src/controllers/authController.js` | 1071 |
| `smartcook-backend/src/controllers/chatController.js` | 784 |
| `smartcook-backend/src/controllers/recipeController.js` | 860 |
| `smartcook-frontend/lib/page/bot_page.dart` | 660 |
| `smartcook-frontend/lib/page/masakan.dart` | 920 |

(Truncated table; full audit only — see source for exact totals.)