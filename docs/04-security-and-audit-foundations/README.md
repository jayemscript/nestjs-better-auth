# Security and Audit Foundations

Release classification: first-release foundation, M1; controls remain mandatory through M5. Last reviewed: 2026-10-15. Status: specification, not a security certification.

[Documentation index](../README.md) · Prerequisites: [Common infrastructure](../02-common-infrastructure/README.md), [Persistence](../03-database-and-persistence/README.md)

## Objective and scope

Enforce one identity/application/assurance policy across the native handler, Nest APIs, BFF calls, and background administration. Record useful security evidence without retaining authentication secrets. Every README sign-in method must pass the same application and MFA authorization gates.

Web and native iOS/Android authentication are mandatory in the first release. The [mobile authentication contract](../14-mobile-authentication/README.md) uses a hosted system browser for primary authentication and enrolled MFA, followed by a proof-bound single-use handoff to a signed opaque session bearer credential.

## Architecture and design

### Trust boundaries and enforcement

| Boundary / threat | Required enforcement |
| --- | --- |
| Internet to BFF: forged origin, credential stuffing, request floods | Same-origin browser mutation checks, bounded bodies, edge rate limits, secure cookies, validated redirects |
| BFF to service: spoofed `x-app-id`, host, forwarding headers | Private authenticated ingress; verify service credential, active service client, scope, bound app, exact public origin; replace untrusted forwarding metadata |
| Native app to public mobile gateway: stolen callback, spoofed app, bundled secret extraction | Treat the app as a public client; exact active platform/return-link registration, S256 PKCE, state and browser correlation, per-transaction/app/IP rate limits; gateway alone holds the app-bound backend service credential |
| Native bearer to resource: token theft, cookie confusion, cached authorization | Keychain/Keystore storage; signed native bearer required; reject mixed cookie/bearer credentials; typed native session resolution and the same authoritative policy checks as web; no cached-profile authorization |
| Bridge-mounted native handler: bypass of Nest guards/interceptors | Bridge `middleware(req,res,next)` route/caller/context policy and native hooks; never assume Nest guards execute for native Express routes |
| Native session to protected operation: pending MFA, stale epoch, bootstrap restrictions | Shared authoritative policy checks native session/account plus app membership, lifecycle, session app binding, current security epoch, bootstrap state, assurance |
| External IdP/SMTP: hostile metadata, callbacks, redirects, relay responses | Protocol-specific verification, bounded egress destinations/timeouts, safe parsing and redaction; no arbitrary URL fetch |
| Storage/operations: excessive grants, leaked secrets, incomplete multi-pool work | Schema roles, reviewed migrations, encryption, durable privileged intent, reconciliation, restricted audit views |

The common policy returns an allow/deny decision with a stable reason, actor, application, assurance state, and permitted capability set. Native identity alone never grants application access. Missing or unavailable policy state denies protected operations. Do not expose raw decision internals to unauthenticated callers.

The selected Nest bridge supplies handler integration and session resolution, not the complete service authorization policy. Disable its automatic global guard and register the explicit composite guard described in [Common infrastructure](../02-common-infrastructure/README.md): native `AuthGuard` plus authoritative identity policy for human routes, separately classified strict service/health checks elsewhere. Native `@Roles` does not replace app permissions, and `@AllowAnonymous` must never be used to bypass service authentication. The bridge middleware and shared native hooks protect native HTTP routes, while server API wrappers protect in-process calls.

Maintain an explicit native endpoint manifest derived from enabled pinned plugins. Classify each route as initial public authentication, native pending-factor completion, service pending-session completion, verified self-service, administration, or protocol callback. Unknown/unclassified routes are denied until reviewed. Route protection must apply before Better Auth side effects; after-hooks alone cannot undo a bypass. Native server `auth.api` calls pass through the same service policy wrapper because they can bypass HTTP middleware and rate limits.

Deny raw HTTP access to native `/api/auth/one-time-token/generate` and `/api/auth/one-time-token/verify`; server-only gateway wrappers enforce transaction, PKCE, application, expiry, replay, and full assurance checks before invoking the native APIs. Configure signed bearer validation (`requireSignature: true`), hashed one-time-token storage, one-minute token expiry, and client generation disabled; never issue a transfer while MFA or bootstrap setup is pending. No raw session credential appears in a callback URL or public native session response.

### Browser origin and callback rules

Use exact HTTPS origins and host-only cookies for approved first-party public hosts. SameSite=Lax is the session/state baseline. Configure Better Auth trusted origins without wildcard acceptance of arbitrary subdomains. Reject unrecognized Host/Origin and spoofed forwarding headers at ingress. BFFs must propagate the original public origin only after validating it against their configured application.

Custom cookie-authenticated mutations require an exact approved Origin, JSON/content type where applicable, and a BFF-issued CSRF token tied to the user's BFF session and checked before forwarding. Validate Fetch Metadata as defense in depth, not as a substitute for the token. Custom GET/HEAD operations are read-only. BFFs do not manufacture a missing browser Origin to turn an unsafe browser request into a trusted one. Service-to-service endpoints use separately authenticated credentials and do not accept cookies as their sole caller authentication.

Keep Better Auth's native origin/CSRF controls enabled. Native protocol callback paths get narrowly reviewed exceptions because legitimate provider redirects or SAML form POSTs may lack the application Origin. Accept only the declared callback method/content type/size; verify native state/nonce/PKCE or SAML request binding, signature, destination, audience, time window, and replay state. Do not apply the generic BFF CSRF token to external IdP callbacks or disable native checks globally. Federation cookie exceptions must be confined to the required native flow with HTTPS and tested against actual browser behavior. [Better Auth security](https://better-auth.com/docs/reference/security)

For SAML ACS, the pinned native source skips its ordinary state-cookie check on cross-site POST because Lax cookies are absent. Native request correlation therefore needs an additional initiating-browser binding. At initiation, the BFF issues a transient host-only Secure/HttpOnly `SameSite=None` correlation cookie and binds its keyed HMAC to provider, native RelayState reference, application, public host and expiry in `identity.auth_flow_bindings`. Extract that opaque RelayState from the approved native authorization URL in the same initiation request; no XML decoding or imaginary AuthnRequest-ID hook is required. Before forwarding POST ACS, locate the binding by RelayState HMAC, verify and atomically consume that browser/application correlation, then retain all native RelayState, InResponseTo, signature and replay checks. Missing, blocked, expired or consumed correlation denies the callback with a safe restart. This narrowly scoped cookie is not a session credential; normal session cookies stay Lax. Do not create a global Origin bypass. The complete flow and source evidence are in [Enterprise SSO](../12-enterprise-sso/README.md).

Safe continuation URLs are built from a server-bound application origin and an allowlisted relative path. Reject alternate schemes, protocol-relative URLs, credentials, mismatched hosts, and paths that escape the approved flow. Shared identities do not authorize cross-application redirects or memberships.

For mobile, the return destination is an exact registered HTTPS Universal Link/App Link, with platform association files and Android signing fingerprints validated at launch and deployment. Use the external system browser, not an embedded WebView. Its authentication origin is a controlled approved first-party origin using the same RP as web, so existing passkeys and browser provider/CSRF controls remain applicable. Link contents contain only a short-lived native one-time token, transaction reference, and state; the token is bound to the initiating S256 verifier and authenticated browser transaction. Never accept a caller-selected return origin or put a bearer/native session token in the URL. The gateway exchange endpoint is intentionally public and rate-limited; a public client identifier is registration context, not a secret or authenticated backend caller. Only its private backend call carries the app-bound service key. A valid link or PKCE proof alone does not waive current membership, lifecycle, security epoch, or enrolled MFA.

Browser cookies retain their existing CSRF and origin policy throughout hosted mobile sign-in. Apply a separate explicit bearer-only resource policy for native app calls; do not create a global missing-Origin exception or globally disable Better Auth's CSRF/origin controls. Strip or reject ambiguous credentials rather than silently preferring cookies or Authorization. Resolve both cookie and signed bearer sessions through typed native `getSession` and then the same service policy.

## Dependencies and prerequisites

Use the shared identity policy, application/service credential binding in [Integration](../13-application-and-service-integration/README.md), and database-backed counters. Reuse the Nest throttler only for custom HTTP routes after verifying its store is shared across replicas; an in-memory per-instance limiter is insufficient as the sole protection. Redis/secondary storage is not a first-release prerequisite.

Native database rate limiting is explicitly enabled in every deployed environment and writes its native table in `auth`. Keep the global baseline at 100 requests/10 seconds per trusted client IP and retain or tighten the pinned plugin rules (for example, credential sign-in and two-factor 3/10 seconds). Do not blindly replace plugin rules with a weaker generic rule. Native `auth.api` calls are outside the HTTP limiter, so wrappers enforce custom policy. [Better Auth rate limiting](https://better-auth.com/docs/concepts/rate-limit)

### Atomic custom abuse policy

Apply the following additional defaults through `identity.abuse_windows`, shared by all replicas and custom/native policy wrappers. Counter key subjects are HMACs with key IDs, not raw emails or IPs. Increment and compare atomically in one database operation/transaction; rejected calls also consume the applicable budget. Return bounded retry guidance without an attacker-controlled permanent account lock.

| Operation | Scope and initial cap |
| --- | --- |
| Password/username credential attempts | Normalized account identity across apps: 10/5 minutes; trusted IP: 30/5 minutes |
| Email challenge issuance/resend | Normalized recipient + application + purpose: 3/15 minutes; trusted IP: 10/minute; one accepted resend per recipient/purpose per 60 seconds |
| Email OTP redemption | Native challenge maximum five failures plus shared account/purpose cap 5/5 minutes; creating a new challenge does not reset shared cap |
| MFA TOTP/recovery verification, including session re-verification | Native user across methods/pending sessions/apps: ten failures/15 minutes; pending flow/session: five failures; trusted IP: 20 attempts/5 minutes; bounded temporary cooldown after failure cap |
| Custom integration validation | Service client: 600/minute initially; ingress aggregate cap sized before launch |
| Mobile transaction creation / exchange | Public native client + trusted IP and transaction where known: 10 starts/minute/IP, 30 starts/15 minutes/application/IP, five invalid exchanges per transaction and 20 exchanges/minute/IP; native server API wrappers apply the same shared counters |

Per-user limits deliberately complement IP limits: rotating pending sessions, sign-in methods, and BFFs must not reset MFA guessing limits. A failure does not change enrolled-factor policy or permanently disable the account. Expired windows release automatically. Release short-lived operation leases on completion, or allow bounded expiry after a crash. Database counter unavailability denies challenge issuance/verification and privileged mutation rather than bypassing limits. High-volume infrastructure exhaustion remains an ingress capacity concern; database counters are not a DDoS service.

The native TOTP verifier does not provide the service's cross-session code-consumption guarantee. After native success and before assurance promotion, atomically insert `identity.mfa_replay_claims(user_id,code_fingerprint,expires_at)` with a 90-second TTL and a unique user/fingerprint key; replace an expired claim atomically. The fingerprint is a keyed HMAC, never an unkeyed six-digit hash, with consistent overlap during key rotation. A live conflict denies promotion even if another native session was created; recovery codes retain native atomic consumption semantics. Native re-verification on an existing session also passes through the custom per-user/per-pending counters because native sign-in attempt caps do not cover that branch. Count factor failures across TOTP/recovery methods and serialize successful verification with factor rotation/current epoch checks. Coordinate these steps through the [MFA](../09-multi-factor-authentication/README.md) orchestration; do not perform a detached replay check after a privileged response has already escaped.

## Database design

Use the [persistence inventory](../03-database-and-persistence/README.md) for `audit.security_events`, `identity.admin_operations`, `identity.abuse_windows`, and operation leases. No duplicate user, password, challenge, or credential store is introduced.

### Security event contract

An event contains `id`, `occurredAt`, validated `requestId`, `event`, `action`, `result` (success/denied/failure/pending), stable `reason`, optional authenticated `actorId`, native `userId`, `applicationId`, target type/ID, durable operation ID, and small allowlisted metadata. IP/UA is restricted metadata, never a primary key or proof of identity. Record unknown callers without inventing a known actor.

Record sign-in/challenge outcomes, issuance throttling, factor enrollment/removal/recovery consumption, session creation/revocation, credential/account changes, linking, membership/role/app changes, bootstrap, SSO trust changes, service-key creation/rotation/revocation, maintenance actions, and reconciliation failures. Record actions once at their semantic owner; hook and controller instrumentation must not create misleading duplicate success events.

Record mobile transaction start, browser completion, code issuance/exchange, replay/proof rejection, expiry, and credential revocation with safe transaction/application references. Exclude PKCE verifiers, callback state, one-time tokens, bearer credentials, browser-binding values, and full verified-link URLs from logs and audit metadata. Failed or uncertain native consumption records a safe failure and requires restart; it cannot be reported as a successful login.

Default retention is 90 days for events and 30 days for restricted IP/UA; a separate restricted maintenance job clears IP/UA at 30 days while preserving event fields, then deletes expired events at 90 days. Metadata allowlists must exclude equivalent identifying network fields so deletion is effective. User deletion replaces unnecessary personal metadata but can preserve pseudonymous event/operation references under the configured policy. Confirm legal/contractual retention before launch rather than inventing a regulatory obligation.

### Audit failure and privileged operations

Before an administrative, membership, factor-reset, SSO-trust, service-key, bootstrap, or account-deletion mutation, persist its operation record and audit intent. If either write fails, do not begin the native/custom mutation. These two writes are independent when audit uses a separate pool; an intent can exist for work that never starts. It is not a distributed transaction.

Execute the guarded mutation, advance the durable phase after each known effect, and append completion or failure evidence. If completion audit fails after a committed change, record the pending phase where possible, alert, and reconcile from observed native/custom state. Do not return a false claim that the committed operation rolled back. An idempotency replay returns the stored result only when the same caller/action/request fingerprint matches; changed input produces conflict.

For ordinary sign-in/denial events, audit failure emits a redacted metric/alert and attempts a bounded retry; it does not turn invalid credentials into success. Successful sign-in still requires durable policy/assurance state. Privileged operations require the durable intent path above. The system cannot guarantee zero audit loss for every library event across independent database failures; operational gaps must be visible.

## API contracts

This foundation adds no public audit-query or security-control endpoint. Feature documents define actual operations and their event names. The internal `SecurityPolicy.evaluate(context, capability)` and `AuditWriter.append(event)` interfaces consume server-derived identity and application context; callers cannot submit their own assurance or admin role.

Custom denials use the shared safe error envelope: 401 invalid caller/session, 403 `APPLICATION_ACCESS_DENIED`/`MFA_REQUIRED`/`BOOTSTRAP_REQUIRED`, 429 `RATE_LIMITED` with `Retry-After`, and 503 `POLICY_UNAVAILABLE` when required dependencies fail. Native errors keep the native shape and native retry headers. Externally visible unknown-account recovery/sign-in errors remain generic; safe internal reason codes distinguish operational cases.

The operational audit reader is a restricted tool/job connection, not an undocumented HTTP endpoint. Do not add a broad audit export route as a side effect of logging work.

## Security considerations

- Never log cookies, Authorization/service keys, passwords, OTPs, TOTP secrets, recovery codes, native session tokens, provider tokens, challenge URLs, SAML assertions, or arbitrary native response bodies. Log route templates and safe request IDs.
- Disable request/response body capture on authentication routes at app, ingress, APM, and error-reporting layers. SMTP protocol debugging is disabled; queue secrets are encrypted.
- Keep app/session/service credentials distinct. A service key authenticates a backend; it does not prove the browser user has signed in, grant a user role, or replace the enrolled second factor.
- Require recent server-recorded second-factor success for privileged actions. Native session freshness and MFA freshness are separate checks.
- Banned/disabled/deleting accounts, revoked service clients, disabled apps, stale epochs, and unbound sessions fail closed even if a cached browser view says otherwise.
- Separate secret-manager access by workload where practical; use versioned key IDs and documented rotation. Querying a user-controlled discovery, metadata, image, attachment, or email-content URL must never provide unrestricted network access.

## Ordered implementation steps

1. Build the trust/route/capability manifest and shared decision interface; inventory native endpoints from pinned source before exposing them.
2. Implement authenticated ingress, exact host/origin/app binding, request-size limits, safe logs, and browser CSRF/callback rules.
3. Enable native database rate limiting and atomic custom counters/leases; propagate trusted IPs and test multi-replica behavior later.
4. Implement account/application/session policy checks before protected native and Nest effects; include server API wrappers and pending-session restrictions.
5. Implement event schemas, field allowlists, append-only privileges, durable privileged intents, and reconciliation.
6. Implement retention, restricted reader access, secret rotation, policy/audit failure alerts, and denial metrics.
7. Exercise the future [security test matrix](../15-testing-strategy/README.md) against every sign-in method before release promotion.

## Measurable acceptance criteria

- SEC-01: Direct native requests and server API calls cannot bypass route classification, application membership, bootstrap, or assurance checks; newly added native routes default to denied.
- SEC-02: Spoofed app/host/proxy headers, invalid BFF credentials, cross-app cookies, hostile origins, unsafe continuation URLs, and missing custom CSRF tokens fail before protected effects.
- SEC-03: Legitimate OAuth/OIDC/SAML callbacks work through their declared exceptions; tampered state, nonce, assertions, destination, or replay fails without a blanket CSRF bypass.
- SEC-04: Parallel calls across replicas/methods/pending sessions cannot exceed atomic custom attempt budgets; window expiry restores availability without permanent attacker-induced lockout.
- SEC-10: The same accepted TOTP cannot promote a second session within the 90-second replay-claim lifetime; parallel successful native verifications produce at most one service promotion, including existing-session re-verification paths.
- SEC-05: Native and custom policy-storage failure never grants protected access; native HTTP and server-only calls have documented limiter coverage.
- SEC-06: Sentinel secrets in URLs, bodies, headers, callback payloads, mail, errors, APM, and ingress capture do not appear in routine logs/events.
- SEC-07: Every privileged mutation has durable pre-operation intent, bounded idempotency behavior, and recoverable completion evidence; injected audit/native/custom commit failures do not imply false atomic rollback.
- SEC-08: Audit runtime cannot read/alter/delete events; restricted reader and retention jobs enforce the 90-day event and 30-day IP/UA defaults.
- SEC-09: Disabled accounts/apps/clients, missing membership, stale security epochs, missing assurance, and insufficiently recent MFA all deny protected operations across every authentication method.
- SEC-11: iOS/Android handoff rejects wrong verifier/state/browser/app/return link, expired or replayed code, intercepted links, mixed cookie/bearer credentials, raw native one-time-token HTTP calls, and pending MFA; no backend service key exists in app bundles.
- SEC-12: Valid signed mobile bearer sessions and web cookie sessions receive the same current policy checks; foreground/resume and backend validation deny revoked/expired sessions even when the app retains a cached profile. Hosted browser sign-in retains normal origin and CSRF enforcement.

## Official references

- [Better Auth security](https://better-auth.com/docs/reference/security)
- [Better Auth rate limiting](https://better-auth.com/docs/concepts/rate-limit)
- [Better Auth hooks](https://better-auth.com/docs/concepts/hooks)
- [Better Auth bearer plugin](https://better-auth.com/docs/plugins/bearer)
- [Better Auth one-time-token plugin](https://better-auth.com/docs/plugins/one-time-token)
- [Better Auth two-factor authentication](https://better-auth.com/docs/plugins/2fa)
- [Nest guards](https://docs.nestjs.com/guards)
- [Selected Nest integration repository](https://github.com/thallesp/nestjs-better-auth)
- [PostgreSQL privileges](https://www.postgresql.org/docs/current/ddl-priv.html)
