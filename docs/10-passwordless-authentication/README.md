# Passwordless Authentication

Release classification: first release, M4. Status: proposed; no authentication code has been implemented. Last reviewed: 2026-10-15.

[Documentation index](../README.md) · Prerequisites: [SMTP](../05-smtp-delivery/README.md), [account lifecycle](../06-authentication-and-account-lifecycle/README.md), [sessions](../07-session-management/README.md), [MFA](../09-multi-factor-authentication/README.md)

## Objective and scope

Provide magic-link and email-OTP sign-in for existing users, plus email verification by OTP. Passwordless authentication proves mailbox access; it does not satisfy the enrolled TOTP/recovery requirement of an MFA-enabled user. Initial registration remains the controlled account-lifecycle flow. Password reset uses the core reset-link flow from phase 06, not the optional OTP password-reset API.

## Architecture and design

Use the bundled `magicLink` and `emailOTP` plugins from the pinned Better Auth release. Configure `disableSignUp: true` on both. Better Auth generates, verifies, expires, and atomically consumes its own challenges. Nest supplies delivery callbacks, application binding, abuse controls, audit, and assurance enforcement. Do not create another OTP generator, challenge table, passwordless session issuer, or mail framework.

Set magic-link `expiresIn: 300` and `storeToken: "hashed"`. Set email OTP `otpLength: 6`, `expiresIn: 300`, `allowedAttempts: 5`, `storeOTP: "hashed"`, and `resendStrategy: "rotate"`. Set core email verification `autoSignInAfterVerification: false`; OTP email verification returns a user update without a new session. These are explicit policy settings, not plugin defaults. The callbacks enqueue encrypted SMTP jobs through phase 05 and await durable enqueue; SMTP submission is asynchronous and expiration-aware.

### Application and scanner binding

Bind issuance to the authenticated BFF's application, exact public origin, purpose, permitted callback paths, and expiry in `identity.auth_flow_bindings`. Store an HMAC of the native token/flow reference, never a second redeemable token. OTP references include the native purpose/email identifier and current issuance generation so resends supersede old application bindings. Serialize send/resend and redemption for the same native identifier using shared PostgreSQL locking; the native plugin's OTP namespace is per purpose/email, not per application. A resend on another application invalidates the earlier code; document that behavior to callers.

For magic links, SMTP content links to a same-origin BFF confirmation page rather than directly navigating to `/api/auth/magic-link/verify`. Its GET renders confirmation only, sets `Cache-Control: no-store` and `Referrer-Policy: no-referrer`, and loads no third-party assets. Explicit user POST with CSRF protection permits the BFF to redeem the original native URL server-side and forward cookies/redirects. The BFF must not prefetch or automatically redeem on page load. Reject direct public GET redemption unless the trusted BFF supplies the in-process/ingress-bound confirmation context; a client header cannot grant this context. Browser implementation is outside this backend documentation deliverable, but this BFF contract is a release prerequisite.

Require the final host and callback to match the issuing application. Opening an application-one link through application two must fail even when both origins appear in `trustedOrigins`. A valid native session produced by OTP/magic link receives `primary_only` assurance and remains unusable for protected native/Nest operations until phase 09 verifies the required factor. A redirect to an MFA page alone is insufficient enforcement.

## Dependencies and prerequisites

Complete M1–M3 first: schema isolation and native atomic verification support, SMTP jobs, controlled enrollment, BFF authentication, active application membership, pending-session enforcement, and shared throttling. Use the native handler owned by the selected Nest integration library and the response-preserving boundary from phase 02. No client SDK, provider email SDK or mail framework is required. BFFs use ordinary same-origin HTTP and forward each cookie separately.

Magic-link and email-OTP authentication are also required for mobile in the first release. Mobile users complete these flows, scanner confirmation and any enrolled MFA inside the approved hosted system-browser portal. Only the guarded [mobile handoff](../14-mobile-authentication/README.md) transfers the resulting authorized session to the device. Email links continue to target the approved HTTPS portal, never a raw reusable bearer credential or arbitrary custom scheme; switching email-reader browsers must require safe restart rather than bypassing transaction/browser binding.

## Database design

Native challenge and session rows live exclusively in `auth`. Generate their schema from the configured plugins and incorporate reviewed SQL into the single TypeORM migration sequence. Do not make TypeORM replicas or subclass native rows from the existing base entity.

Reuse `identity.auth_flow_bindings`, `identity.session_assurance`, and `identity.email_deliveries`; use `audit` for events. Flow binding and delivery are separate transactions from native challenge creation. An enqueue failure returns failure and leaves any existing challenge unauthorized by missing/failed service binding; cleanup later removes expired rows. Never report delivery success merely because a native token was created. On a partial session/assurance write failure, deny protected access and revoke or expire the orphan native session as described in phase 07.

## API contracts

The following are verified native v1.7.8 methods, prefixed with `/api/auth`. Native JSON errors retain their library shape and cookies; do not wrap them in the Nest envelope. Every initiation requires an authenticated approved BFF, exact origin, body validation, and abuse budget. Every redemption additionally needs an unexpired matching application flow binding. Returned native `token`/session fields are sensitive, remain at the BFF boundary, and never become log/audit fields.

| Native method and path | Request / caller | Success and side effects | Errors and audit |
| --- | --- | --- | --- |
| `POST /sign-in/magic-link` | `email`; optional native `name`, `callbackURL`, `newUserCallbackURL`, `errorCallbackURL`, `metadata`. Service permits only approved callback paths and bounded non-security metadata; no registration fields are authoritative. | `200 {status:true}`; native hashed challenge, bound flow, SMTP enqueue. Existing/unknown email gets the same outward initiation result; suppress dispatch for unknown users without revealing existence. | Native malformed-email/validation errors; custom policy denies wrong app/origin; enqueue failure is retryable failure. Audit `passwordless.requested` with method/purpose and pseudonymous subject, never URL/token. |
| `GET /magic-link/verify` | Native `token`, optional `callbackURL`, `errorCallbackURL`, `newUserCallbackURL`; reached only through explicit BFF confirmation. | Consumes native token; creates native session/cookie; redirects to approved callback. Without callback the native result is `{token,user,session}`; release-one BFF supplies the bound callback and does not expose that raw body. Initializes assurance. | Invalid/consumed/expired/wrong-purpose token redirects with native `INVALID_TOKEN`; unknown user with disabled signup reports `new_user_signup_disabled` through the safe error callback. Audit `passwordless.failed` or `sign_in.primary_succeeded`. |
| `POST /email-otp/send-verification-otp` | `email`, `type` restricted by service to `sign-in` or `email-verification`; verification issuance is tied to the current user's email or controlled registration context. | `200 {success:true}`; rotates challenge and queues SMTP for existing user. Unknown user gets the same success body without delivery. | Native `INVALID_EMAIL`; unsupported purpose rejected before native handler; queue failure/throttle as above. Audit `passwordless.requested` or `email_verification.requested`. |
| `POST /sign-in/email-otp` | `email`, `otp`; native also accepts optional `name`, `image` and additional signup fields, which the service rejects because signup is disabled. | `200 {token,user}` plus cookie; consumes sign-in OTP, marks proven email as verified where applicable, creates a native session and pending assurance. Native promotion can revoke unproven account access. | `400 INVALID_OTP` / `OTP_EXPIRED`, `403 TOO_MANY_ATTEMPTS`; unknown account after challenge proof still cannot register. Audit failed attempt or primary success and email-verification transition. |
| `POST /email-otp/verify-email` | `email`, `otp`; bind to matching current user or registration verification flow. | `200 {status:true,token:null,user}` with auto sign-in disabled; consumes email-verification OTP and updates native email verification. | Native invalid/expired/attempt-limit errors; `USER_NOT_FOUND` after OTP proof. Audit `email_verification.succeeded` or failure; do not imply MFA success. |

Block all other plugin HTTP routes in the native allowlist: `POST /email-otp/check-verification-otp`, `POST /email-otp/request-password-reset`, deprecated `POST /forget-password/email-otp`, `POST /email-otp/reset-password`, `POST /email-otp/request-email-change`, and `POST /email-otp/change-email`. The `createVerificationOTP` and `getVerificationOTP` functions are server-only in this version; they are not exposed HTTP retrieval endpoints, and hashed OTPs cannot be read back in plaintext. Do not document the deprecated comment's GET for `check-verification-otp`: its actual route declaration is POST.

No new Nest endpoint is introduced in this phase. Shared service policy failures use phase 02 error meanings; native errors/statuses and redirect responses remain native. The confirmation page/POST is an application BFF contract, not an invented Better Auth endpoint.

## Security considerations

Rate-limit by application, normalized-mailbox fingerprint, IP/network, and native challenge purpose. Atomic per-identifier counters/locks must prevent parallel failures and resends from resetting abuse budgets. A resend rotates the native challenge, but does not reset the service's rolling abuse budget. Return generic issuance responses and avoid timing/account-existence signals; never send a link for a disabled or inaccessible account merely because the plugin can generate one.

Consume flow bindings once alongside authorization of redemption; a failed redemption must not unlock a consumed native token. Database failures deny success. Keep challenges, OTP bodies, delivery ciphertext/plaintext, callback query strings, and native token responses out of telemetry. SMTP retries never extend native expiry. Mailbox proof never grants application membership, roles, SuperAdmin status, or MFA assurance. Recovery does not turn these methods into a second factor.

## Implementation plan

1. Configure the pinned plugins with the explicit lifetimes, hashing, rotation, and disabled-signup settings; review the resulting native migration SQL.
2. Add exact exposed routes and purpose restrictions to the shared native policy boundary. Connect existing SMTP enqueue callbacks without inline relay calls.
3. Add issuance-generation/application binding and shared locking around OTP send/resend/redeem; bind magic callbacks before dispatch.
4. Implement the BFF confirmation contract and trusted redemption context; preserve native redirects/cookies and prevent scanner GETs from reaching consumption.
5. Route every resulting native session through membership and phase 09 assurance gates. Keep email verification non-signing-in.
6. Produce the future acceptance evidence below before promoting M4; update the documented contract if the pinned library changes.

## Acceptance criteria

- PWL-01: Existing users sign in through magic link and email OTP; unknown users cannot auto-register and initiation does not disclose account existence.
- PWL-02: OTPs are six digits, expire after five minutes, rotate on resend, and lock after five failures; concurrent attempts and resends cannot bypass rolling limits.
- PWL-03: Native OTP/token values are hashed at rest; delivery payloads are encrypted, never logged, and not sent after expiry.
- PWL-04: Expired, replayed, wrong-purpose and concurrently redeemed challenges grant at most one usable session; database failures fail closed.
- PWL-05: A scanner GET does not consume a magic link; explicit same-origin confirmation does. Wrong-host/cross-application callback and binding swaps fail.
- PWL-06: MFA-enabled users cannot use any protected native or Nest mutation following passwordless primary authentication until TOTP/recovery succeeds; disabled users and missing memberships remain denied.
- PWL-07: Email-verification OTP updates the intended user's verified state without creating a session; optional OTP reset/change/check HTTP routes and plaintext retrieval are unavailable.

## Official references

- [Magic-link plugin](https://better-auth.com/docs/plugins/magic-link) and [v1.7.8 implementation](https://github.com/better-auth/better-auth/blob/v1.7.8/packages/better-auth/src/plugins/magic-link/index.ts)
- [Email-OTP plugin](https://better-auth.com/docs/plugins/email-otp), [v1.7.8 routes](https://github.com/better-auth/better-auth/blob/v1.7.8/packages/better-auth/src/plugins/email-otp/routes.ts), and [v1.7.8 options](https://github.com/better-auth/better-auth/blob/v1.7.8/packages/better-auth/src/plugins/email-otp/types.ts)
- [Better Auth security](https://better-auth.com/docs/reference/security) and [two-factor authentication](https://better-auth.com/docs/plugins/2fa)

Route declarations and executable return values above were inspected statically. Runtime concurrency, BFF confirmation, SMTP failure behavior, and integration compatibility remain future verification work.
