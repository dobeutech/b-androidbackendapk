# Monetization Readiness Plan — androidbackendapk (Enterprise MDM)

Status: **pending approval** — not yet executed.

## Requirements Summary

- **Deployment model**: Enterprise MDM. Company-owned/managed devices, employees informed as a
  condition of employment, distributed via managed Google Play (private app) rather than the
  consumer Play Store review track. This rules out the stalkerware/covert-monitoring policy
  problem the permission set would otherwise raise, but does not eliminate all Play policy review
  (see Risks).
- **Timeline**: aggressive — weeks, not months. Fix only what blocks charging a real enterprise
  customer real money. Defer polish, full docs, full test coverage, and anything not on the
  critical path to a first paid contract.
- **Web dashboard**: deferred entirely to v2. MVP ships as Android agent + Firebase backend +
  Stripe billing; admins operate via a lightweight internal tool or the Firebase console until
  v2.
- **Goal**: move from a partially-broken, never-shipped prototype (0 revenue) to a state where one
  enterprise customer can be onboarded, billed via Stripe, and supported.

## Acceptance Criteria (testable)

1. `firebase-backend/functions` builds cleanly (`npm run build` / `tsc --noEmit` exits 0) from a
   single canonical API implementation — not two divergent `index.js`/`index.ts` entrypoints.
2. `android-app/gradlew build` succeeds locally and in CI (wrapper committed to git).
3. The Android app installs, runs, and reports at least one real metric to Firestore end-to-end on
   a physical device or emulator.
4. Stripe is wired: a company can be created, subscribed to a paid plan via Stripe Checkout, and
   the assigned tier is written to Firestore only after a webhook-verified payment event —
   `updateSubscription` can no longer grant a plan without payment.
5. The release Android build is signed with a real release keystore, not the debug key.
6. A single canonical Android package exists (the `agent`-namespaced tree); the orphaned flat
   duplicate is removed or archived.
7. A minimal legal package exists: Privacy Policy, Terms of Service, and a one-page data-processing
   summary suitable for enterprise procurement review.
8. The existing test suite (2 Android unit tests, 3 instrumented tests, 1 backend integration test)
   passes in CI, plus new unit tests for the billing-gate logic in `subscription-service.js`.
9. `docs/ARCHITECTURE.md` and `docs/OPERATIONS.md` exist, covering: how a new enterprise customer
   is provisioned, how billing works, and how to deploy.
10. `CHANGELOG.md` exists and `versionCode`/`versionName` are bumped to reflect the first sellable
    release.

## Implementation Steps

### Phase 0 — Triage & canonicalize (unblocks everything else)

1. Pick the canonical backend: the TypeScript Express app
   (`firebase-backend/functions/src/index.ts` + `api/auth.ts` + `api/devices.ts`) over the plain-JS
   `index.js`, since it's the more maintainable/testable structure. Port any logic that's only in
   `index.js`, then delete it and update `package.json`'s `main`.
2. Create the missing `middleware/` files `index.ts` already imports but that don't exist —
   `auth.ts`, `rateLimit.ts`, `errorHandler.ts`, `requestLogger.ts` — under
   `firebase-backend/functions/src/middleware/`, so the backend actually compiles.
3. Resolve the duplicate Android source trees: the manifest namespace is
   `com.androiddiagnostic.agent`, so that tree is canonical — confirm nothing references the flat
   `com.androiddiagnostic.*` tree, then delete or archive it.
4. Generate and commit the Gradle wrapper (`gradlew`, `gradlew.bat`,
   `gradle/wrapper/gradle-wrapper.properties`) at a version compatible with AGP 8.1.0 — every CI
   workflow already assumes it exists.
5. Add `.gitattributes` (`* text=auto eol=lf`) per the portfolio's repo onboarding gate 1.

### Phase 1 — Real billing (the actual "monetized" part)

6. Implement Stripe Checkout + a webhook handler in a new
   `firebase-backend/functions/src/services/stripe-service.ts`: a `createCheckoutSession`
   endpoint, and a `stripeWebhook` endpoint that verifies `stripe.webhooks.constructEvent` and
   updates Firestore only on `checkout.session.completed` / `customer.subscription.updated`.
7. Lock down `updateSubscription` in the canonical API so plan changes only happen via the
   verified webhook or an authenticated admin-only override — remove the current unauthenticated
   direct-write path in `subscription-service.js`.
8. Add/verify Firestore security rules so a client cannot write `planId` directly
   (`firebase-backend/firestore.rules`).

### Phase 2 — Ship-readiness

9. Add a real release signing config in `android-app/app/build.gradle` (new keystore, secrets via
   `local.properties`/CI secrets, never committed).
10. Bump `versionCode`/`versionName`; add `CHANGELOG.md`.
11. Fix CI now that the wrapper exists (`android-build.yml`, `ci.yml`,
    `integration-tests.yml`); remove or keep-guarded the `web-dashboard` CI steps since it's
    deferred.
12. Write `docs/ARCHITECTURE.md` and `docs/OPERATIONS.md` by mining the existing root-level
    planning docs (`APK_MIGRATION_PLAN.md`, `FIREBASE_BACKEND_SPEC.md`,
    `IMPLEMENTATION_SUMMARY.md`, etc.) rather than starting from scratch.
13. Write a minimal `SECURITY.md` and a real `.env.example` (Stripe keys, Firebase project vars —
    placeholders only).
14. Draft a Privacy Policy, ToS, and a one-page data-processing summary tailored to the enterprise
    MDM model (company-owned devices, employee notice, retention periods pulled from the existing
    `subscription-service.js` retention-days fields).

### Phase 3 — Verify

15. Run the full test suite (Android unit + instrumented, backend integration against the Firestore
    emulator); fix anything the canonicalization work broke.
16. Manual smoke test: register a test company → subscribe via Stripe test mode → enroll one
    Android device → confirm metrics flow to Firestore → confirm plan-gated enforcement works
    (e.g., hitting the device limit blocks further enrollment).

## Explicitly deferred (out of scope for this pass)

- Web dashboard, in any form — v2.
- Amplitude/analytics instrumentation — a real portfolio-policy gap, but not a monetization
  blocker; track separately (see Verification Steps).
- Full 80% test-coverage target — only fix what's broken or directly touched by this work.
- Twilio/SendGrid (SMS/email) flows — dependencies exist but are unused; Stripe handles receipts
  natively, so leave for v2 unless a specific need appears.
- `legacy/` directory — already marked deprecated, leave untouched.

## Risks & Mitigations

- **Play distribution policy risk**: even a managed-Google-Play private app goes through review,
  and the accessibility service / background location / `DUMP` / `READ_LOGS` permissions may still
  trigger a restricted-permission review even for an enterprise app. *Mitigation*: start the Play
  Console permission-declaration process now, in parallel with engineering — it has external lead
  time that will otherwise become the critical path. Separately, evaluate whether Android's
  official Device Policy Controller / Android Management API can replace the more invasive
  permissions (accessibility service, root manager) — flag as a fast-follow architecture review,
  not a blocker for this pass.
- **Stripe integration surface**: there is zero existing Stripe code today, so a fully hardened
  webhook implementation (retries, idempotency, proration/downgrade edge cases) doesn't fit a
  "weeks" timeline. *Mitigation*: scope to the minimum — Checkout + webhook + idempotent Firestore
  write — and explicitly defer proration/downgrade UX polish.
- **Duplicate backend/Android trees**: risk of deleting the wrong copy and losing logic.
  *Mitigation*: diff both before deleting anything, and grep the Android app's Retrofit interfaces
  to confirm which backend contract the client actually calls.
- **No test coverage on `subscription-service.js` today**: billing-logic changes without tests risk
  revenue-affecting bugs. *Mitigation*: add targeted unit tests for `canAddDevice`,
  `canExecuteCommand`, and `hasFeature` as part of Phase 1, even though full coverage stays
  deferred.
- **CI is very likely fully broken today** (no committed wrapper) — until Phase 0 step 4 lands,
  there is no CI signal at all; verify everything locally first.

## Verification Steps

- `cd firebase-backend/functions && npm run build` exits 0.
- `cd android-app && ./gradlew build` exits 0 (after the wrapper is committed).
- `cd firebase-backend/functions && npm test` (against the Firestore emulator) passes, including
  new Stripe/webhook tests.
- Manual Stripe test-mode end-to-end flow (Phase 3, step 16).
- Run `node repos/_standards/checks/check-amplitude.mjs androidbackendapk` and record the verdict
  — even a deliberate "defer" decision should be on record, not just unassessed.
