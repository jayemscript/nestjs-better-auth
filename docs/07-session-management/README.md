# Session Management

Release classification: first-release requirement, M2; all-method assurance completed in M3/M4. Last reviewed: 2026-10-15.

[Documentation index](../README.md) · Prerequisites: [Core identity](../06-authentication-and-account-lifecycle/README.md), [Persistence](../03-database-and-persistence/README.md), [Security](../04-security-and-audit-foundations/README.md). Consumers: [MFA](../09-multi-factor-authentication/README.md), [Integration](../13-application-and-service-integration/README.md).

## Objective and scope

Use authoritative, database-backed Better Auth sessions for web and native iOS/Android sign-in, renewal, logout, visibility, and revocation. Web uses cookies; mobile uses the signed opaque native bearer value obtained after complete hosted-browser authentication and guarded handoff. Add token-free session views and application/MFA policy. No JWT, access/refresh pair, independent session issuer, or client-side success cache is introduced.

## Architecture and design

Set session expiry to seven days, update age to one day, and native freshness to five minutes. Disable cookie caching, secondary authentication storage, and stateless sessions. These are explicit service settings, not inferred from a cookie's expiry. Renewal uses the native library and maintains the same policy binding; arbitrary HTTP traffic must not extend a session through a custom updater. [Native session model](https://better-auth.com/docs/concepts/session-management).

Each application proxies native routes under its own exact public origin and receives host-only Secure/HttpOnly cookies. Native sessions are global identity records, but each newly issued session is bound to one application in `identity.session_assurance`. An opaque cookie copied to another app is rejected by policy even when native authentication succeeds. Native callback/state cookies use their reviewed protocol-specific settings. Do not configure a parent-domain session cookie to simulate cross-app SSO.

The native session lookup identifies a user and may update expiry. Application authorization additionally reads the bound application, membership, account lifecycle/ban, bootstrap state, account security epoch, and assurance. Read these on every protected request, including native mutation routes and internal validation. Missing/stale assurance or policy storage failure never becomes a successful authorization. Do not trust `x-app-id`, browser role fields, or `get-session` alone.

### Session states and transitions

| State | Entry and permitted use | Exit |
| --- | --- | --- |
| No session | Anonymous credentials/provider initiation; native pending password-MFA cookie is not a session | Native successful issuance |
| `primary_only`, MFA enrolled | Native non-credential sign-in issued a session; only minimal `/me`, approved factor completion/recovery, and logout | Native TOTP/recovery success marks the resulting session verified; pending expiry revokes it |
| `primary_only`, no MFA enrolled | Normal user primary sign-in; normal access requires active account/membership and completed bootstrap | Enrolling MFA advances security epoch and requires the documented assurance transition |
| `verified` | Native TOTP/recovery success bound to this session/app/epoch | Revocation, expiry, epoch change, or account/membership restrictions invalidate use |
| Bootstrap restricted | Either assurance state, but password change/TOTP enrollment is incomplete | Ordered bootstrap checks in 08; no normal access until active |

Pending sessions expire after five minutes even though the native maximum session lifetime is longer. A privileged operation additionally requires factor success in the previous five minutes. Native freshness is based on session creation; recent factor verification does not magically make an old native session fresh. Require a new sign-in when the native route requires freshness. Reject trusted-device bypass. A credential-based MFA flow can have no session until the factor completes; create assurance from the final successful issuance, not the pending response.

When native password changes, MFA completion, or other operations replace a session, hooks resolve the resulting session from the native response/context or persisted issued session. Do not attach assurance only to the old incoming `ctx.session`. Verify the concrete hook ordering against the pinned version and refuse issuance/protected access when the identity write fails. Reconciliation can revoke an orphan native session but cannot infer that MFA succeeded from its mere existence.

## Dependencies and prerequisites

Reuse Better Auth core session APIs through the selected Nest integration library and shared policy/audit services. Configure `session.cookieCache.enabled=false`, `session.expiresIn=604800`, `session.updateAge=86400`, and `session.freshAge=300`; omit secondary storage. Enable the native bearer plugin with `requireSignature=true`; the common resolver/library guard passes trusted Authorization or Cookie headers through native `getSession`, rejecting mixed transports. Native sessions remain in `auth`. Backend integration calls the service over HTTP as specified in 13; consumers never query the auth database directly.

The [mobile contract](../14-mobile-authentication/README.md) governs one-time transfer to Keychain/Keystore storage. Transfer preserves the same session ID, original method/provider, expiry, MFA timestamp, epoch and app binding. A token is not a refresh token or offline authorization. Strip native bearer-export headers from web browser responses; emit mobile credentials only through the guarded exchange. Foreground/resume and protected calls revalidate authoritative policy, and online logout/revocation invalidates mobile and web on the next request.

## Database design

Native session IDs/tokens/user IDs/timestamps remain library-owned. Tokens are bearer-equivalent secrets even though a signed cookie normally carries them. TypeORM never manufactures or changes native expiry/token fields.

`identity.session_assurance` records native session ID, user ID, application ID, security epoch, primary method, state, optional MFA method/time, pending expiry, and version. It is not a second session store and contains no token. Application/account policy rows determine current access. Session views join the native records and assurance server-side; no cross-schema FK assumes atomic deletion.

Native revocation precedes assurance cleanup. If native removal succeeds but custom cleanup fails, access remains revoked and reconciliation deletes the orphan policy row. Epoch changes first invalidate assurance across applications, then session revocation completes; a failed revoke cannot restore access. Cleanup periodically removes assurance for expired/missing sessions. Membership suspension blocks authorization immediately even when the underlying shared identity session remains valid for another permitted operation.

## API contracts

Native routes below require the authenticated BFF/application context. `N` means exact native response/errors and cookies. The library's native session object can include a token; do not wrap it into general identity responses, cache it, or log it.

| Method/path | Owner/caller/input | Result | Errors, side effects, audit |
| --- | --- | --- | --- |
| GET `/api/auth/get-session` | Native core; cookie optional, current app; supported query `disableCookieCache?`, `disableRefresh?` | N: `{session,user}` or `null`; informational only | Native invalid-cookie/storage errors or null behavior preserved; may renew/update cookie unless disabled; metadata-only lookup event |
| POST `/api/auth/sign-out` | Native core; cookie, no body; allowed for pending/restricted users | N: `success:true`, optional provider logout `url,redirect`, and cleared cookies | Native errors preserved; durably invalidate current assurance before native revocation attempt; `session.signed_out` with revocation outcome |

Expose only GET for native `/get-session`, although v1.7.8 also defines POST. The native routes below are documented server-only operations, not externally reachable browser APIs. The in-process operation capability from 06 permits their typed calls only after policy checks. This avoids exposing bearer tokens merely to let users select a device.

| Native method/path, not externally exposed | Typed input and native output | Wrapper responsibility |
| --- | --- | --- |
| GET `/api/auth/list-sessions` | Current cookie; native session array; requires fresh native session | Filter/project all owned application sessions; native freshness still applies |
| POST `/api/auth/revoke-session` | `{token}`; `{status:true}` | Resolve owned target session ID to token in server memory; never accept token from client |
| POST `/api/auth/revoke-other-sessions` | Current cookie, no body; `{status:true}` | Preserve current session while removing all other owned app sessions |
| POST `/api/auth/revoke-sessions` | Current user's valid cookie, no body; `{status:true}` | Revoke that authenticated user's sessions only; not an arbitrary-target recovery/admin API |

Recovery uses the native configured `revokeSessionsOnPasswordReset` path. Administrative target revocation uses the admin plugin's typed `revokeUserSessions({userId})` with its required authorized admin context. Current-user `revokeSessions` requires that user's still-valid cookie and cannot be repurposed by submitting a target user ID. Choose the documented native operation for the actual actor, and invalidate service policy before any failure-prone revocation continuation.

Proposed custom endpoints use 02's envelope. Complete assurance means active membership/account, current epoch, satisfied bootstrap, and enrolled factor satisfied. Listing and revocation require a five-minute native session; factor freshness is additionally required when MFA is enrolled. This intentionally matches native list freshness rather than hiding its errors behind an inconsistent custom policy.

| Method/path | Caller/input | `data` and status | Errors, side effects, audit |
| --- | --- | --- | --- |
| GET `/api/v1/me/sessions` | Complete fresh session, current app membership; `limit?,cursor?` | 200 page of `SessionView + {application:{id,name}}` for all owned active sessions across apps | 401 invalid; 403 `SESSION_NOT_FRESH`/policy; 503 dependency failure. No tokens or unowned sessions; `session.listed` |
| DELETE `/api/v1/me/sessions/:sessionId` | Same policy; opaque owned native session ID; no body | 204, empty | 404 unknown/foreign ID; shared policy errors. Native revocation then assurance cleanup; clear cookies if current; `session.revoked` with target app/ID |
| POST `/api/v1/me/sessions/revoke-others` | Same policy; no body | 200 `{revoked:true}` | Shared policy/native storage errors; revokes all other owned app sessions, preserves current; `session.others_revoked` |

`SessionView` is defined in 02. Derive a coarse escaped `deviceLabel` from user agent rather than rendering raw HTML; an IP summary is optional and redacted, not location proof. Return no provider credentials or session token. `lastSeenAt`, when available, is explicitly approximate from native update activity, not a claim that every request persists activity. A cursor traverses a stable `(createdAt,id)` ordering over the current native list; concurrent expiry/revocation can shorten a page without exposing another user's records.

Selecting a session uses its native record ID, never a URL containing its token. Resolve ownership before resolving token and invoke the native API with trusted caller headers; zero results and foreign IDs share 404. Repeated deletion of a now-missing ID is 404 rather than an invented second successful operation. Internal session validation is defined once in 13 and applies these same guards; native `/get-session` is not its substitute.

All response families use `Cache-Control: no-store`. Forward every native `Set-Cookie` separately through the BFF, preserve expiry/deletion attributes, and never join cookies into a comma-separated header. A failed identity join returns 503 or denial, never a native-authentication-only success through a custom endpoint.

## Security considerations

Use native cookie signing and rotation. Authentication secrets live in approved configuration; deploy a compatible key/secret rotation sequence from 16. Ban, account disabling/deletion, security-epoch changes, revoked membership, and missing assurance deny immediately on the next request, including long-lived sessions. Do not cache positive policy decisions in a BFF during release one.

Current-session logout remains available to pending/bootstrap users and should not require a fresh session or completed factor. It still uses native origin/CSRF behavior. Versioned native sign-out catches database deletion failures and can return success after clearing the cookie. The service before-hook must therefore durably remove/invalidate the current assurance first, and track any native deletion still needing retry. Missing assurance denies protected access and is never recreated from an existing session. If that durable invalidation is unavailable, report a safe policy 503 rather than claim confirmed revocation. An already absent session remains an idempotent cookie-clear case. A server-side revocation does not clear another browser's cookie instantly; the next use fails because the authoritative session no longer exists or assurance has been invalidated.

Separate native expiry, service pending expiry, native creation freshness, and recent MFA time. Never advance any of them from client timestamps. Native sign-in result tokens are sensitive protocol data; only custom views are approved for general-purpose frontend state. Device names/IPs are display hints and cannot authorize a session.

## Implementation plan

1. Configure explicit session/cookie settings and ingress app binding; reject unknown hosts and strip spoofable forwarding context.
2. Implement the shared authoritative resolver and policy join. Wire it into native hooks, custom guards, and backend validation.
3. Persist assurance on actual native issuance, handling credential MFA pending flows and rotated sessions. Add pending expiry and orphan reconciliation.
4. Expose only the reviewed native lookup/logout routes. Deny raw native token-based session administration at the external boundary.
5. Add token-free views and ID-based revocation wrappers with ownership checks, native freshness, cross-app labels, redacted audit, and cookie forwarding.
6. Connect recovery, ban, deletion, role/security changes, and MFA transitions to epochs/revocation. Ensure partial failures remain denied and retryable.
7. Collect the following evidence during later implementation; no runtime checks are run by this documentation task.

## Acceptance criteria

- SES-01: A new native session persists in `auth`, has the configured seven-day/one-day behavior, and uses a host-only Secure/HttpOnly cookie without a cookie-data cache or parallel JWT.
- SES-02: Native lookup preserves nullable/native output and headers; application access is denied when membership, app binding, lifecycle, epoch, or assurance is invalid even if lookup itself finds a native session.
- SES-03: Copying one app's cookie/session to another approved app fails policy; the BFF forwards all cookie headers correctly and sends no auth responses to shared caches.
- SES-04: Session views list only the owner's active records, including app labels, with no token fields, raw secrets, or unescaped device data; freshness requirements match the contract.
- SES-05: ID-based single-session and all-other revocation work across owned applications; foreign IDs fail, current-session deletion clears cookies, and revoked sessions fail their next authoritative lookup.
- SES-06: Logout works for pending/bootstrap sessions. Expired MFA pending sessions are revoked and cannot mutate credentials, enroll factors, link providers, or call protected Nest/native routes.
- SES-07: Credential MFA and every session-rotation flow attach assurance to the resulting issued session; missing policy writes fail closed and cannot be reconstructed as verified from a cookie alone.
- SES-08: Database/policy outages never authorize; native revocation plus failed custom cleanup stays revoked, epoch invalidation plus failed native revocation stays denied, and native sign-out's swallowed database failure cannot leave the old assurance usable. Reconciliation completes outstanding cleanup.
- SES-09: iOS/Android signed bearer sessions obey the same native expiry, app/epoch/MFA checks and revocation as cookies; raw unsigned native tokens, mixed transports and offline cached-profile authorization are rejected.

## Official references

- [Better Auth session management](https://better-auth.com/docs/concepts/session-management)
- [Cookie settings](https://better-auth.com/docs/concepts/cookies)
- [Options](https://better-auth.com/docs/reference/options)
- [Versioned session API and middleware source](https://github.com/better-auth/better-auth/blob/v1.7.8/packages/better-auth/src/api/routes/session.ts)
- [Versioned sign-out source](https://github.com/better-auth/better-auth/blob/v1.7.8/packages/better-auth/src/api/routes/sign-out.ts)
