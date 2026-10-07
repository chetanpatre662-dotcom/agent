# FitTrack — Forensic Audit: Auth-on-reopen + FCM/Firestore rewrite-every-login

Audit only. No code was changed in this phase. Every claim below cites the actual
current code (file + function + line context) that was read during this audit.

## Scope read
Frontend: `lib/main.dart`, `lib/app.dart`, `lib/core/services/firebase_init.dart`,
`lib/core/providers/app_providers.dart`, `lib/firebase_options.dart`,
`lib/features/auth/data/auth_repository.dart`, `lib/features/auth/providers/auth_providers.dart`,
`lib/features/auth/providers/account_provider.dart`, `lib/features/auth/providers/auth_controller.dart`,
`lib/core/routing/app_router.dart`, `lib/core/routing/route_names.dart`,
`lib/core/services/fcm_service.dart`, `lib/core/services/notification_service.dart`,
`lib/features/notifications/notification_providers.dart`, `lib/features/shell/home_shell.dart`,
`lib/core/services/api_client.dart`, `pubspec.yaml`, `pubspec.lock`.
Backend: `src/controllers/authController.ts`, `src/services/authService.ts`,
`src/repositories/userRepository.ts`, `src/middleware/auth.ts`,
`src/controllers/notificationController.ts`.

SDK versions (pubspec.lock): `firebase_core 3.15.2`, `firebase_auth 5.7.0`,
`firebase_messaging 15.2.10`, `cloud_firestore 5.6.12`.

---

## Traced lifecycle timelines

### Cold reopen (second launch, session persisted on device)

```
main() → WidgetsFlutterBinding.ensureInitialized()
       → SharedPreferences.getInstance()
       → FirebaseInitializer.initialize()  (awaits Firebase.initializeApp — returns BEFORE
                                             the native SDK finishes restoring the session)
       → NotificationService.init()         (local notifications only; NOT FCM)
       → runApp(ProviderScope(...))
app.dart → watch appRouterProvider → GoRouter(initialLocation '/')
app_router redirect runs on first frame:
       firebaseReady == true
       authStateProvider == AsyncLoading   → redirect holds on '/' (splash)   ✅ correct so far
       │
       │  Native Firebase restores persisted token via a platform-channel callback.
       │  authStateChanges() emits its FIRST event. Per FlutterFire this first event
       │  can be a transient `null` immediately before the restored user, OR the
       │  restored user directly, depending on timing.
       ▼
   _deferLeadingNull sees first event:
       - if first event == restored user  → forwarded immediately → AsyncData(user) → /home ✅
       - if first event == null           → deferred one Future.microtask; guarded by
                                            _auth.currentUser != null.
             - if currentUser already populated when the microtask drains → null suppressed ✅
             - if currentUser NOT yet populated when the microtask drains  → null FORWARDED
                                            → authStateProvider = AsyncData(null)
                                            → redirect: signedIn == false → /login  ❌ BUG
```

The last branch is the auth symptom: a logged-in user is routed to `/login` on reopen.

### First login (fresh sign-in)

```
LoginScreen → AuthController.login() → AuthRepository.login() → signInWithEmailAndPassword
       → authStateChanges() emits AsyncData(user) (post-startup, forwarded immediately)
       → accountInfoProvider runs → POST /api/auth/verify
            backend: authenticate middleware verifies token → verify controller
                     → authService.verifyAndSync → userRepository.ensureAccount
                        → doc exists? set(patch,{merge:true}) with updatedAt=serverTimestamp()  ← WRITE every time
       → router: verified + onboarded → /home → HomeShell.initState
       → postFrame: _initServices() → fcmServiceProvider.init()
            → requestPermission + getToken() → POST /api/notifications/register-token  ← WRITE every time
                 backend: addFcmToken → arrayUnion(token) + updatedAt=serverTimestamp()  ← WRITE every time
```

Every path that reaches `/home` (login AND successful reopen) runs the identical
`HomeShell.initState` → `FcmService.init()` → `getToken()` → register sequence. That is
symptom #2.

---

## AUTH ROOT CAUSES

### ROOT CAUSE #1 — Leading-null race is not fully closed by `Future.microtask`
- **File/function:** `lib/features/auth/data/auth_repository.dart` → `_deferLeadingNull` (and its
  use in `authStateChanges()`); consumed by `authStateProvider` in
  `lib/features/auth/providers/auth_providers.dart`.
- **What happens:** The leading `null` from `authStateChanges()` is deferred by a single
  `Future.microtask`, which forwards it unless either a superseding user event has arrived OR
  `_auth.currentUser != null` at the moment the microtask drains.
- **Why it happens:** Firebase native session restore is delivered on Android via a
  **platform-channel callback**. Platform-channel results are posted to the Dart event loop as
  **a future task, not necessarily the current microtask drain**. A `Future.microtask`
  scheduled during the first frame can therefore drain in an **earlier** event-loop turn than
  the turn that both (a) delivers the restored-user stream event and (b) populates
  `_auth.currentUser`. When that ordering occurs, neither guard is satisfied, so the null is
  forwarded and `authStateProvider` becomes `AsyncData(null)`. The in-code comment claims
  "Microtasks drain in the same event-loop turn as the Firebase platform-channel callback" and
  that "The native Firebase SDK updates FirebaseAuth.currentUser synchronously" — neither is
  guaranteed by the SDK. `currentUser` is only populated **after** restore completes, which is
  exactly the window being guarded.
- **Why it appears on reopen:** The race only exists during the cold-start restore window.
  After startup, `signInWithEmailAndPassword` populates state synchronously, so fresh logins
  are unaffected — the user only sees it when reopening with a persisted session, which matches
  the report "login required on every reopen."
- **Confirmation re: Case A vs Case B:** Case A (Firebase persists; router/provider mishandles
  the restore window) is confirmed. There is no Firebase-persistence failure in code: single
  `Firebase.initializeApp` path (`firebase_init.dart`), no `setPersistence` override, no
  alternate local auth source of truth (grep for `isLoggedIn|SecureStorage|Hive|authToken`
  found nothing).
- **Proposed minimal fix:** Make the router gate on an explicit "initial auth state settled"
  signal instead of racing a microtask. `firebase_auth 5.7.0` exposes
  `FirebaseAuth.instance.authStateReady()` (a Future that completes once the initial auth state
  is settled — the restored user or a genuine null). Preferred fix: introduce an
  `authReadyProvider` (FutureProvider awaiting `authStateReady()`), and in the router treat auth
  as **loading until `authStateReady()` completes**; after it completes, seed from
  `_auth.currentUser` synchronously and only then interpret `null` as signed-out. This removes
  the need for `_deferLeadingNull` to guess ordering. (Alternative, if staying stream-only: in
  `authStateChanges()` emit an initial synchronous seed from `_auth.currentUser` **after**
  awaiting `authStateReady()`, then forward the stream.) Keep the "distinguish AsyncLoading from
  AsyncData(null)" behavior already present in the router (see #3) — it is correct and must be
  preserved.

### ROOT CAUSE #2 (secondary / confirm-correct) — Router state handling is correct and should be preserved
- **File/function:** `lib/core/routing/app_router.dart` → `redirect` and `appRouterProvider`.
- **What happens / why it is correct:** The redirect **does** distinguish `AsyncLoading` from
  `AsyncData(null)`:
  - `if (authAsync.isLoading) return splash;` holds on splash while loading (does NOT send to
    `/login`) — so there is **no** "redirect to /login while auth is loading" branch. (Answers
    audit question 3: no such branch exists; the only `/login` redirect is under
    `if (!signedIn)` which runs only after `isLoading` is false.)
  - `accountInfoProvider` loading also holds on splash; `accountInfoProvider` **error** routes to
    `/startup-error` (session preserved, retry/sign-out) — it does **not** log the user out.
    (Answers audit question 4: an account error/timeout does NOT cause logout; `/startup-error`
    preserves the session. Sign-out there is user-initiated only.)
  - `refreshListenable` is driven from `ref.listen` on the **same** providers the redirect reads
    (`authStateProvider`, `accountInfoProvider`) with `fireImmediately: true`, which fixed the
    earlier "two independent subscriptions" race.
- **Why #1 still bites despite this:** The router correctly waits during `AsyncLoading`, but once
  `_deferLeadingNull` forwards the transient null the state becomes `AsyncData(null)` — a
  *settled* value the router must honor as signed-out. The defect is upstream in the repository
  (#1), not in the router. **Do not** add router hacks; fix the stream/readiness gate.
- **Proposed minimal fix:** None to the router logic itself beyond consuming the new
  `authReadyProvider` from #1 (treat not-ready as loading). This section documents that the
  router is otherwise sound.

### signOut() hunt — result: CLEAN
Every `signOut()`/`logout()` occurrence was traced:
- `auth_repository.dart` `logout()` → `_auth.signOut()` — called only by `AuthController.logout()`.
- `auth_repository.dart` `deleteAccount()` → `_auth.signOut()` after a successful backend delete.
- `AuthController.logout()` callers: `profile_screen.dart` (Log out tile), `verify_email_screen.dart`
  ("Use a different account"), `onboarding_screen.dart` (Sign out), `app_router.dart`
  `_StartupErrorScreen` (Sign out button). **All user-initiated.**
- **No** `signOut()` is triggered by backend-verify failure, timeout, 401, FCM failure, or
  Firestore failure. `accountInfoProvider` throws on failure → router shows `/startup-error`
  (session preserved). **No violating path found.** A temporary backend/network/FCM/Firestore
  failure cannot destroy a valid Firebase Auth session.

---

## FCM ROOT CAUSES (symptom #2 — frontend side)

### ROOT CAUSE #3 — FCM `getToken()` + backend register runs unconditionally every time `/home` is reached
- **File/function:** `lib/features/shell/home_shell.dart` → `_HomeShellState.initState` →
  `addPostFrameCallback` → `_initServices()` → `ref.read(fcmServiceProvider).init()`; and
  `lib/core/services/fcm_service.dart` → `FcmService.init()`.
- **What happens:** On **every** build of `HomeShell` (i.e., every time routing lands on `/home`
  — every cold start that reaches home AND every fresh login), `FcmService.init()` calls
  `messaging.getToken()` and then `_registerToken(token)` → `POST /api/notifications/register-token`
  **with no comparison** against a locally cached token or a backend-known token. It also
  attaches a fresh `onTokenRefresh` listener each time (another unconditional register on
  rotation). (Answers audit questions 1 and 3: yes, unconditional; yes, tied to reaching the
  authenticated home shell on every restore/login.)
- **Why it happens:** `FcmService` holds `_token` only in instance memory; `fcmServiceProvider`
  is a plain `Provider` recreated per app run, so there is no persistent "already registered this
  token" check. `getToken()` returning the same value still triggers a backend write.
- **Why it appears on every login/startup:** `initState` of the home shell fires on every entry
  to `/home`, so the register call fires every launch and every login even when the token is
  unchanged.
- **`deleteToken()` check (audit question 2):** `deleteToken()` is **never** called anywhere.
  `FcmService.unregister()` calls the backend `DELETE /api/notifications/token` but
  `unregister()` has **no caller** in the repo — it is not wired to logout. So no forced token
  rotation on startup/login. Good (no delete-on-startup bug), though logout does not unregister.
- **Proposed minimal fix:** Persist the last-registered token (e.g. in SharedPreferences) and
  skip the `register-token` POST when `getToken()` returns the same value already registered.
  Register only on actual change: first-ever token, or a value differing from the cached one,
  plus the `onTokenRefresh` path. Attach the `onTokenRefresh` listener once per app run (guard
  against re-subscribing on repeated `HomeShell` builds).

---

## FIRESTORE-WRITE ROOT CAUSES (symptom #2 — backend side)

### ROOT CAUSE #4 — `ensureAccount` rewrites `users/{uid}.updatedAt` on EVERY `/api/auth/verify`
- **File/function:** `backend/src/repositories/userRepository.ts` → `ensureAccount` (the
  existing-doc branch); reached from `authController.verify` → `authService.verifyAndSync`.
- **What happens:** For an existing user, it unconditionally executes
  `set({ email, emailVerified, updatedAt: serverTimestamp() }, { merge: true })`. Because
  `updatedAt` is set to `serverTimestamp()` every call, this is a Firestore write on **every**
  verify — i.e., every login and every startup that calls verify — even when `email` and
  `emailVerified` are unchanged.
- **Why it appears on every login/startup:** `accountInfoProvider` calls `/api/auth/verify` on
  every authenticated resolve, so the doc's `updatedAt` moves every launch. This is the
  "user document updated on each launch even when nothing changed" report.
- **createdAt / identity (audit question re: new identity):** `createdAt` is set **only** in the
  not-exists branch and is **never** rewritten — confirmed correct. The doc id is the Firebase
  UID (`this.col.doc(uid)`), so the same UID always maps to the same `users/{uid}` doc; **no new
  identity is created on reopen** — confirmed, not a bug. Only `updatedAt` (and redundant
  email/emailVerified) churn.
- **Proposed minimal fix:** In the existing-doc branch, compare incoming `email`/`emailVerified`
  against the stored snapshot and **only** write (and only then bump `updatedAt`) when a field
  actually changed. If nothing changed, skip the write and return the existing data.

### ROOT CAUSE #5 — `addFcmToken` always writes `updatedAt`, so register-token writes even when the token is unchanged
- **File/function:** `backend/src/repositories/userRepository.ts` → `addFcmToken`; reached from
  `notificationController.registerToken`.
- **What happens:** `set({ fcmTokens: arrayUnion(token), updatedAt: serverTimestamp() }, {merge:true})`.
  `arrayUnion` is idempotent for the array contents, but the write itself (and `updatedAt`) is
  executed on **every** call regardless of whether `token` is already present.
- **Why it appears on every login/startup:** Combined with #3 (client registers unconditionally
  on every `/home` entry), the backend performs a Firestore write on every launch/login,
  bumping `updatedAt` and re-touching the doc.
- **Proposed minimal fix:** Make the endpoint a no-op when the token is already in `fcmTokens`:
  read the doc (or use a transaction) and only `arrayUnion` + bump `updatedAt` when the token is
  new. The primary cost reduction comes from fixing #3 (don't call the endpoint when unchanged);
  this is defense-in-depth so a repeat call doesn't churn `updatedAt`.

---

## Backend auth correctness (confirmed, not bugs)
- `middleware/auth.ts` verifies the ID token with an 8s timeout, derives UID from the verified
  token only, does not use `checkRevoked`, and maps a verification *service* error to 503 (not
  401) — so a transient verify stall won't masquerade as "logged out." Correct.
- `/api/auth/verify` and `verifyAndSync` never create a second identity; UID→doc mapping is
  stable. Correct.

## Local storage / multiple init (confirmed, not bugs)
- Single `Firebase.initializeApp` call site (`firebase_init.dart`).
- No competing auth source of truth: no `isLoggedIn`, `SecureStorage`, `Hive`, or cached auth
  token found. `SharedPreferences` is used only for app prefs (launch motivation, etc.).
- SDK versions are current and support the recommended fix (`authStateReady()` is available in
  `firebase_auth 5.7.0`). No version issue is implicated.

---

## Summary — is `Future.microtask` enough?
No. It fixes the *common* ordering (restore event lands in the same or earlier turn than a
macrotask `Timer`) but does **not** guarantee correctness under AOT when the platform-channel
restore callback lands in a **later** event-loop turn than the microtask drain and
`_auth.currentUser` is not yet populated. The robust fix is to gate the router's readiness on
`FirebaseAuth.authStateReady()` and seed synchronously from `currentUser` once ready, rather
than deferring/guessing. The router's AsyncLoading-vs-AsyncData(null) handling is already correct
and should be kept.

Two bugs are independent and both must be fixed:
- **Auth (reopen → login):** Root Cause #1 (repository readiness race). Router is sound.
- **FCM/Firestore churn (every login):** Root Causes #3 (client unconditional register), #4
  (`ensureAccount` unconditional `updatedAt`), #5 (`addFcmToken` unconditional `updatedAt`).

Sources: [Firebase — Manage users in Flutter](https://firebase.google.com/docs/auth/flutter/manage-users),
[Firebase JS API reference (authStateReady semantics)](https://firebase.google.com/docs/reference/js/auth.auth).
Content was rephrased for compliance with licensing restrictions.
