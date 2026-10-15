# Authentication and Account Lifecycle

Release classification: complete first-release requirement, M2; advanced methods join in M4. Last reviewed: 2026-10-15.

[Documentation index](../README.md) · Prerequisites: [Infrastructure](../02-common-infrastructure/README.md), [Persistence](../03-database-and-persistence/README.md), [Security](../04-security-and-audit-foundations/README.md), [SMTP](../05-smtp-delivery/README.md). Read the minimal application/BFF provisioning contract in [13](../13-application-and-service-integration/README.md) before exposing authentication traffic.

## Objective and scope

Provide verified email/password and username/password authentication, account profile management, verification, password recovery, explicit account linking, and durable deletion. Better Auth is the sole credential and native challenge authority. Shared user identities never imply access to every application.

## Architecture and design

Create one Better Auth instance with core email/password, the username plugin, and the policy hooks required by later plugins. Supply it through `@thallesp/nestjs-better-auth` AuthModule/AuthService; the integration library owns the native handler at `/api/auth` using the composition order in 02. Do not mount a parallel Node handler. Add custom account views and orchestration through Nest; never recreate credential verification in a Nest controller.

These same authentication methods must serve web and native iOS/Android in release one. Mobile performs registration, credential entry, verification/recovery and factors in its approved system-browser hosted portal, then receives a signed native session through the guarded [mobile handoff](../14-mobile-authentication/README.md). Existing native endpoint bodies remain browser-hosted; no static secret or second credential protocol is shipped in the device application.

Configure `emailAndPassword.enabled=true`, `requireEmailVerification=true`, `autoSignIn=false`, `revokeSessionsOnPasswordReset=true`, and a 15-minute reset lifetime. Set verification to one hour, send on sign-up, and disable automatic sign-in after verification. A successful registration/verification is not an authenticated session. Keep Better Auth's native scrypt password handling; proposed policy is 12–128 characters, supporting spaces and password managers without composition rules or silent truncation. No bcrypt/Argon2/JWT subsystem is added just because packages already exist. [Email/password options](https://better-auth.com/docs/authentication/email-password).

Keep email normalization and uniqueness in the native implementation. Username is optional for accounts created by other methods; when provided it is 3–32 ASCII letters, digits, `_`, or `.`, normalized to lowercase by the plugin, and unique across the shared realm. Disable the separate display-username field for a smaller public contract. Permit username changes through the guarded profile operation and native uniqueness checks. Do not expose the optional username-availability endpoint, which is unnecessary for sign-in and creates an enumeration surface. [Username plugin](https://better-auth.com/docs/plugins/username).

Registration is allowed only for an active ingress-bound application with `self_registration=true`. On actual native user creation, persist its account policy and a member assignment for that application. Additional application access requires explicit membership approval. Never create membership from a synthetic duplicate-sign-up response. If native creation succeeds but policy/membership creation fails, deny access and reconcile from durable operation context; do not guess which application to grant on a later login.

Use core email callbacks only to enqueue the encrypted SMTP job in 05. Enable `user.changeEmail.enabled`, configure `sendChangeEmailConfirmation` for the existing address, and keep `updateEmailWithoutVerification=false`; new-address verification uses the ordinary verification callback. Do not open SMTP connections in controllers or issue custom reset/verification tokens. Verification and recovery redirects must resolve to approved paths on the originating application's exact public origin. Bind flow context server-side using `identity.auth_flow_bindings`; a link redeemed at another approved app is still invalid for that flow.

Set `account.accountLinking.enabled=true`, `disableImplicitLinking=true`, `allowDifferentEmails=false`, and `allowUnlinkingAll=false`. Keep forced trusted-provider linking disabled. Linking requires an already authenticated, verified user, a fresh native session, recent enrolled MFA, and proof from the selected provider. A matching email in an unsolicited provider login must not merge accounts. See [users/accounts](https://better-auth.com/docs/concepts/users-accounts) and the provider-specific restrictions in [11](../11-social-authentication-and-passkeys/README.md) and [12](../12-enterprise-sso/README.md).

Serialize credential removals across all method types with one per-user credential-mutation advisory lock and recoverable operation lease. Recheck the complete usable sign-in/recovery set under that lock, then retain it through native unlink/passkey deletion and the corresponding identity epoch update. Separate per-route checks are insufficient: concurrent OAuth unlink and passkey deletion could otherwise each observe the other method and remove both. On a cross-pool partial failure, keep conflicting removal blocked until reconciliation establishes the actual remaining methods.

### Policy and native-response boundary

Every entry route uses an explicit method/path allowlist. This phase exposes only the native routes in its table, plus the session routes in 07 and subsequent explicitly reviewed methods in 09–12. Unknown routes fail closed. Deny external raw `/admin/*`, SSO management, `/api-key/*`, token-export/refresh/account-info endpoints, `set-password`, and native deletion/session-management HTTP endpoints. The `/refresh-token` core account endpoint concerns external provider tokens, not a second service-session refresh API.

Trusted typed server API calls are authorized through a non-serializable operation context, such as AsyncLocalStorage, established after Nest guards. A browser cannot grant this authority by submitting a header or body field. Native success/errors, headers, cookies, and redirects are preserved. A service policy denial is an intentional exception: native hooks throw a safe native-compatible APIError with a stable service code; Nest routes use the shared custom envelope. Neither can be mistaken for a successful native protocol result.

MFA-enabled users remain pending until the enrolled factor succeeds, including when a provider directly created a native session. Missing account policy or session assurance denies protected access. The native `get-session` response is informational and never proves membership or MFA. All protected native profile, credential, linking, and enrollment mutations use the same policy as Nest. See [09](../09-multi-factor-authentication/README.md).

## Dependencies and prerequisites

Use the reviewed Better Auth 1.7.8 baseline, the existing Express/Nest adapter, native PostgreSQL persistence, username/admin plugins, and SMTP callbacks. Complete database grants, durable audit intent, abuse controls, exact application binding, and the minimal provisioning path before opening registration. No provider SDK or mail framework is required.

The future implementation must enable and validate native PostgreSQL transactions for token consumption, password reset, and session revocation. This document records source inspection, not executed compatibility or concurrency evidence.

## Database design

Better Auth owns `auth` user, account, session, and verification records and their native identifier formats. Credential hashes live in the native credential account, never in `identity`. Username adds its plugin-owned user field/index. Native verification records and signed-token handling must retain their actual library behavior; do not promise every verification mechanism uses an identically hashed database token.

TypeORM owns `identity.account_policies`, initial `application_memberships`, `auth_flow_bindings`, `session_assurance`, and `admin_operations` as defined in 03. Native user IDs are opaque string references without cross-schema foreign keys. Audit events use the append-oriented contract in 04. Native and TypeORM pools are separate transaction boundaries.

### Durable deletion and recovery

Use a staged deletion operation rather than treating `afterDelete` as a transaction:

1. Validate self-service proof or administrative authority, last-SuperAdmin protection, and idempotency. Persist operation intent before destructive work.
2. In native `beforeDelete`, resolve the trusted operation context; set lifecycle `deleting`, advance the security epoch, disable access, and revoke all native sessions. The authorized deletion continuation is the only permitted operation for that context.
3. Continue through Better Auth's native user deletion, preserving its plugin/table cleanup. A native failure leaves a disabled account and retryable operation; never reactivate automatically.
4. In `afterDelete`, remove memberships, assurance, flow bindings, pending mail payloads, and service-owned personal data. Preserve the minimal tombstone needed for operation reconciliation.
5. Pseudonymize identifying audit references under the retention policy in 04; retain security facts and operation outcomes. Mark completion only after required cleanup finishes.

For an administrator's typed `removeUser` call, run the same preparation/finalization explicitly: do not assume it invokes core `deleteUser` callbacks. A reconciler detects native absence, completes idempotent cleanup, and reports stuck operations. No cross-pool atomicity or rollback of deleted credentials is claimed. Protect audit immutability through the narrowly authorized retention/pseudonymization path, not arbitrary updates by runtime actors.

Recovery sets a new native password and revokes every existing session across applications. The post-reset callback advances the account epoch, invalidates assurance/flows, and emits audit evidence. Native reset may already be committed when this callback fails; reconcile the missing policy work and require a new login rather than reporting the old password as restored. MFA remains enrolled; email recovery does not bypass the second factor.

## API contracts

`N` below means native Better Auth output/error semantics, with no Nest envelope. Native user/session results are sensitive and may contain a session token. Send `Cache-Control: no-store`; exclude bodies from logs/analytics and never persist them in browser storage. Body fields with `?` are optional. All browser calls require the authenticated, exact-origin BFF binding from 13; public means no end-user session, not direct unauthenticated access to the private service.

For native errors, preserve the library status/body; typical failures are malformed input 400/422, bad credentials 401, verification/policy denial 403, and throttling 429. Service policy denials use safe codes including `APPLICATION_ACCESS_DENIED`, `MFA_REQUIRED`, `BOOTSTRAP_REQUIRED`, and `POLICY_UNAVAILABLE` (503), in the route family's error shape. Audit names below are proposed stable event families; include success/failure and redacted context, never credentials/challenges.

| Method and full path | Owner/caller and input | Successful result | Failure, side effects, audit |
| --- | --- | --- | --- |
| POST `/api/auth/sign-up/email` | Native core + username; public self-registration app; `name,email,password,username?,callbackURL?` | N: `user,token:null`; generic synthetic equivalent for existing email | Native validation/duplicate behavior; reject privileged extra fields. Actual creation adds native credential + policy/member and queues verification; `account.registration` |
| POST `/api/auth/sign-in/email` | Native core; public; `email,password,rememberMe?,callbackURL?` | N: credential sign-in result or native MFA pending result | Wrong credentials/unverified email/ban; creates session or native pending challenge; `authentication.primary` |
| POST `/api/auth/sign-in/username` | Native username; public; `username,password,rememberMe?,callbackURL?` | N: credential result or native MFA pending result | Native username/password/verification errors; same assurance gates as email; `authentication.primary` |
| POST `/api/auth/send-verification-email` | Native core; public; `email,callbackURL?` | N: `status:true` | Native request validation and resend throttling; queues existing native challenge; `challenge.verification_requested` |
| GET `/api/auth/verify-email` | Native core; public native flow; query `token,callbackURL?` | N: verification result or redirect | Invalid/expired token, wrong flow binding; marks verified without automatic sign-in; `account.email_verified` |
| POST `/api/auth/request-password-reset` | Native core; public; `email,redirectTo?` | N: generic `status,message` | Native validation/rate limits; creates reset challenge + queued SMTP; `challenge.password_reset_requested` |
| GET `/api/auth/reset-password/:token` | Native core; public native flow; query `callbackURL` | N: redirect to approved reset UI | Invalid/expired token redirects with native error; no password change; `challenge.password_reset_opened` |
| POST `/api/auth/reset-password` | Native core; public native flow; `newPassword,token` in body | N: `status:true` | Invalid/expired/reused token or password policy; resets credential and revokes all sessions; `account.password_reset` |
| POST `/api/auth/update-user` | Native core + username; active membership and complete assurance; `name?,image?,username?` only | N: `status:true` | Validation/username conflict, denied server-only fields; updates native profile; `account.profile_updated` |
| POST `/api/auth/change-password` | Native core; fresh authenticated session, complete assurance, or the narrow owned `password_change_required` bootstrap state; `currentPassword,newPassword,revokeOtherSessions:true` | N: native `user,token` result; rotation headers preserved | Wrong current password, freshness/policy failure; replaces password, revokes other sessions; `account.password_changed` |
| POST `/api/auth/change-email` | Native core; fresh complete session; `newEmail,callbackURL?` | N: `status:true` | Collision/invalid proof/redirect; sends current-address approval then new-address verification before final change; `account.email_change_requested` / `account.email_changed` |
| GET `/api/auth/list-accounts` | Native core; complete active session; no body | N: linked-account list | Invalid session/policy; read only; redacted account-access event |
| POST `/api/auth/link-social` | Native core; fresh complete session and verified email; `provider:"google",callbackURL?,errorCallbackURL?` | N: provider redirect/URL result | Unsupported provider, mismatched email/proof, unsafe redirect; bound linking flow; `account.link_requested` / `account.linked` |
| POST `/api/auth/unlink-account` | Native core; fresh complete session; `accountId` is native account record ID | N: `status:true` | Foreign account/last usable method/recovery-risk denial; unlinks selected native account; `account.unlinked` |

This allowlist deliberately narrows the library's optional input surface: reject caller-supplied roles, verification flags, arbitrary provider scopes/tokens, or unreviewed plugin fields. `rememberMe=false` may request a shorter native cookie session; it never extends the maximum session policy. Native change-email callbacks use SMTP for both existing-address confirmation and new-address verification. Keep privileged/bootstrap accounts' identity changes subject to the administration safeguards in 08.

The bootstrap password-change exception requires the exact owned bootstrap run/user, verified email, original application binding, fresh native session, current password proof, and `bootstrap_state=password_change_required`. It permits only that password replacement and transition to `mfa_enrollment_required`; it grants no other credential mutation or normal access. It does not apply to an already MFA-enrolled pending user or to arbitrary recovery requests.

Core verification links in the pinned version are signed JWT email proofs, not identically stored/consumed opaque tokens like magic links or OTPs. Preserve native behavior: a repeated still-valid verification may return an idempotent already-verified result/redirect, but must not issue a session, grant new membership, or repeat an actual email-change side effect. Tampered/expired proofs fail. Application bindings and change-email transition checks remain required; do not claim every native verification link is inherently single use.

Custom APIs use 02's envelope and cookie/CSRF policy. These are proposed service routes, not library endpoints.

| Method and path | Caller/input | `data` and status | Errors, side effects, audit |
| --- | --- | --- | --- |
| GET `/api/v1/me` | Current native session bound to this app; no body | 200 `{user:UserView,currentAppRole,assurance:{state,mfaVerifiedAt?},bootstrapState}` | 401 invalid; 403 missing membership/disabled lifecycle; 503 policy unavailable. Pending MFA/bootstrap may read this minimal status view only. No tokens, enrollment secrets, or role enumeration; read audit metadata |
| DELETE `/api/v1/me` | Fresh complete session, active app, recent MFA if enrolled; `Idempotency-Key`; body `{currentPassword?}` required for a credential account | 202 `{operationId,state:"pending"|"running",lifecycle:"deleting"}` after durable preparation; forwards native cookie clearing | 400 invalid password; 409 last SuperAdmin; normal shared errors. Invokes typed `deleteUser` with native `password` proof; saga above; `account.deletion_requested/completed/failed` |

Deletion is asynchronous from the client's perspective even if native deletion completes quickly. A repeated request with the same key and actor/body replays its stored result; a changed payload is 409. The client clears local authenticated UI on 202. Typed native calls made by a custom wrapper must request headers/Response explicitly and forward every cookie mutation; the plain typed data return does not carry cookie headers to Nest automatically. Operation progress is available to authorized administrators through 08; a revoked/deleted account cannot poll protected APIs. Do not expose native `/delete-user` or its token callback as an alternate bypass.

## Security considerations

Use shared anti-enumeration responses for email registration/recovery and shape synthetic signup plugin fields consistently with real output. Verify this with the admin/username plugins enabled. The callback enqueue and unknown-user paths must also avoid meaningful status/timing differences, including a delivery-queue outage; a generic success body alone is insufficient. Preserve the native protocol while adding consistent boundary availability checks and throttling rather than accepting lost delivery. Username conflicts are necessarily observable when selecting a public handle; throttle attempts and avoid an availability service. Reject unknown profile fields at the service policy boundary even when native schemas allow plugin extension fields.

Configure sign-in, verification, resend, recovery, linking, and deletion limits separately using 04. Do not add fixed request sleeps or await SMTP to equalize timing. A durable enqueue may be awaited; delivery occurs later. Verification/recovery URLs are secrets; redact their query/path tokens everywhere, including Better Auth's configured logger.

Keep provider tokens encrypted using the supported native option where available, or paired native storage/read hooks after version review; do not apply write-only encryption that breaks later native reads. No provider access-token export endpoint is exposed. Reject unlinking the last usable method and require recovery coverage before removing an enrolled factor or passkey. A password change that rotates sessions must create assurance for the resulting session from verified server context rather than copying browser flags or an obsolete `ctx.session` reference.

## Implementation plan

1. Complete prerequisite modules and minimal app/BFF provisioning. Configure one native instance, explicit routes, native logger redaction, safe callbacks, and PostgreSQL transaction behavior.
2. Enable core credentials and username rules. Apply native schema migrations through the single reviewed migration stream. Add registration policy hooks, actual-create membership initialization, and partial-creation reconciliation.
3. Connect verification/reset/change-email callbacks to SMTP jobs; configure lifetimes, no auto sign-in, and approved redirect/flow binding.
4. Add the shared native/Nest policy checks. Create account policy/assurance at native session creation and resolve the final rotated session after successful native operations.
5. Expose the guarded profile, account-linking, and minimal current-user contracts. Deny sensitive unused native routes before enabling plugins.
6. Enable all-session reset revocation and post-commit identity cleanup/reconciliation. Require `revokeOtherSessions:true` for ordinary password changes and preserve the resulting native cookie/token response.
7. Implement durable self/admin deletion with native callbacks and operation context. Exercise failures between every pool boundary during the later testing task.
8. Collect AUTH acceptance evidence alongside sessions and MFA before advancing beyond M3; M2 alone is not the complete release.

## Acceptance criteria

- AUTH-01: Email signup creates one native credential identity and only the originating approved membership; duplicate email produces the configured generic shape without granting access or revealing the real user ID. Known/unknown email challenge requests have reviewed status/body/timing equivalence when queue insertion succeeds and when it fails.
- AUTH-02: Email and normalized username login succeed for verified users; wrong passwords and unverified/banned users fail. Username case collisions cannot create two identities.
- AUTH-03: Verification is delivered through the SMTP job boundary, expires after one hour, rejects wrong-app redirects/flow bindings, and never auto signs in.
- AUTH-04: A reset challenge expires after 15 minutes; concurrent/replayed redemption cannot change the password twice. A rejected new password follows the actual native token-retention behavior.
- AUTH-05: Successful reset invalidates every prior application session; failures after native commit reconcile without claiming that credentials rolled back or disabling MFA.
- AUTH-06: Native result shapes, status, redirects, and every cookie are preserved; no parallel password/token store exists, and arbitrary role/verification fields cannot be written through registration/profile routes.
- AUTH-07: Explicit verified linking works; implicit same-email merging, wrong-email linking, foreign account unlinking, and removal of the last usable method fail, including concurrent mixed-method removals serialized through the shared credential lock.
- AUTH-08: Profile/password/email changes require the documented policy; email changes require the existing-address and new-address proofs, and password rotation binds assurance to the resulting session.
- AUTH-09: Deletion disables access before cleanup, revokes all sessions, removes native/plugin and custom personal records, pseudonymizes retained audit references, survives every partial failure, and cannot delete the last eligible SuperAdmin.

These are future implementation gates. No tests, builds, installations, or generators were run for this documentation task.

## Official references

- [Better Auth email/password](https://better-auth.com/docs/authentication/email-password)
- [Username plugin](https://better-auth.com/docs/plugins/username)
- [Users and accounts](https://better-auth.com/docs/concepts/users-accounts)
- [Hooks](https://better-auth.com/docs/concepts/hooks)
- Versioned route source: [signup](https://github.com/better-auth/better-auth/blob/v1.7.8/packages/better-auth/src/api/routes/sign-up.ts), [sign-in](https://github.com/better-auth/better-auth/blob/v1.7.8/packages/better-auth/src/api/routes/sign-in.ts), [password](https://github.com/better-auth/better-auth/blob/v1.7.8/packages/better-auth/src/api/routes/password.ts), [email verification](https://github.com/better-auth/better-auth/blob/v1.7.8/packages/better-auth/src/api/routes/email-verification.ts), [profile/deletion](https://github.com/better-auth/better-auth/blob/v1.7.8/packages/better-auth/src/api/routes/update-user.ts), and [accounts](https://github.com/better-auth/better-auth/blob/v1.7.8/packages/better-auth/src/api/routes/account.ts)
