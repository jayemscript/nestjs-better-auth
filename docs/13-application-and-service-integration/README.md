# Application and Service Integration

Release classification: first-release requirement; initial caller provisioning and ingress are M1 prerequisites, full integration is M5. Last reviewed: 2026-10-15.

[Documentation index](../README.md) · Related: [Core identity](../06-authentication-and-account-lifecycle/README.md), [Authorization](../08-authorization-and-bootstrap/README.md), [MFA](../09-multi-factor-authentication/README.md)

## Objective and scope

Provide stable HTTP integration for standalone, monolithic, SaaS, and microservice callers without distributing database access or authentication signing secrets. The first release must support both first-party web applications through BFFs and native iOS/Android applications through a public mobile gateway. All user authorization remains scoped to an application even though users share a realm. The mandatory [mobile authentication contract](../14-mobile-authentication/README.md) specifies system-browser sign-in, full MFA, verified return links, PKCE-bound one-time handoff, signed opaque bearer sessions, and secure device storage.

The minimal application/service-key provisioning path is implemented with phase 04 before authentication traffic is enabled. It is documented here to keep its full contract in one place. Do not wait until M5 to authenticate ingress, and do not require an already authenticated human to provision the first BFF.

## Architecture and design

### Public and private topology

Browsers call their own origin, such as `https://one.apps.example.com`. That application proxies only the approved `/api/auth/*` and `/api/v1/*` paths to this private backend and injects its service credential. Provider callbacks traverse that same proxy. Native mobile clients use the public gateway's transaction/exchange and bearer resource interfaces; they never receive the gateway's service key. The mobile hosted-browser portal follows the same protected browser/native-handler path. `/internal/v1/*`, health details, and metrics are never exposed to browser or device clients.

Use exact allowed production hosts inside a controlled parent domain, for example `one.apps.example.com` and `two.apps.example.com`. Shared WebAuthn RP ID is `apps.example.com`. These values are examples; deploy with actual controlled origins. Use native allowlisted dynamic base URL resolution with HTTPS and no unknown-host fallback. A BFF's authenticated application must match its asserted public host, current origin, callback allowlist, and any session binding. Global `trustedOrigins` alone does not enforce this per-app boundary.

Host-only cookies keep sessions independent per application. Shared identities and inbound enterprise SSO do not imply a browser silently shares an auth-service cookie between applications. Broad parent-domain cookies and arbitrary unrelated-domain SSO are not first-release assumptions.

### Caller authentication

Adopt Better Auth's API Key plugin for service credentials. Use a dedicated configuration for BFF/backend credentials, hashed key storage, explicit expiration, permission metadata, and `enableSessionForAPIKeys: false`. Each service client has a dedicated native service user, marked `kind=service` in account policy. These identities cannot sign in through browser password, OTP, passkey, or provider routes and cannot gain human admin roles.

Ingress reads `X-Service-Key` and invokes the typed server-only key-verification API. Verification must check plugin key validity and required permissions, then join only through the service-owned key-to-client mapping and active application. The plugin result's key ID/reference must match the mapped service user/client. Missing mapping, revoked client, or a scope outside the client's approved scope set fails closed. Do not enable API-key-to-user-session conversion.

`x-app-id` is optional consistency metadata. If present it must equal the authenticated client's application. A browser cannot select a new app by changing a header. Strip any browser-supplied service key and application/forwarding assertions at the BFF; replace them from server configuration. Private-network placement alone does not authenticate a caller.

The private backend receives a native cookie from the web BFF or a signed opaque bearer session from the mobile gateway, plus that server caller's service key. Internal validation carries exactly one declared session transport through a server-to-server body, never a URL or log; it is not a JWT introspection endpoint. Service credentials identify the server caller, and native sessions identify humans. Neither replaces the other. Reuse the selected `@thallesp/nestjs-better-auth` AuthModule/AuthService and common policy resolver; do not mount a second direct handler.

### Proxy contract

| Concern | Required behavior |
| --- | --- |
| Upstream | Fixed auth backend destination; never accept a user-controlled proxy URL |
| Path | Preserve approved path and query; deny `/internal`, raw native `/api/auth/api-key/*`, and unknown native paths; guarded custom `/api/v1/admin/service-clients` routes remain reachable only through their declared human-authorized policy |
| Host/protocol | Replace with registered public origin context; trust forwarding only from authenticated configured ingress |
| IP | Strip untrusted forwarded chains; append the resolved client IP under the known network topology |
| Body | Preserve raw auth stream/content type; enforce bounded streaming size and timeout |
| Session transport | Web/hosted-browser flows forward required auth cookies and preserve separate `Set-Cookie` headers; mobile resource calls forward signed bearer only and reject mixed credentials. Strip bearer-export headers from browser responses |
| Redirects | Forward native status/location without following on the server; validate destination through native and per-app policy |
| Cache | `no-store` for auth and identity responses, including errors; no CDN cache of authenticated responses |
| Retry | No automatic retry of sign-in, token redemption, factor verification, or non-idempotent mutations |

### Session validation and authorization

Consumers call the internal session-validation API for every request that requires authoritative identity. The service performs native database session lookup with cookie cache disabled, verifies session/application binding, membership, lifecycle, security epoch, and MFA/bootstrap policy. No successful authorization caching is enabled initially.

Ordinary application resource permissions remain the downstream application's responsibility. This service supplies identity and app-level permissions; it does not decide ownership of an order, invoice, or other external domain object. Internal authorization checks accept only the exact registered identity-service permission catalog in [08](../08-authorization-and-bootstrap/README.md), retaining its own-user/target/application restrictions.

Revocation takes effect on the next authoritative check. Requests already authorized/in flight may finish; long-lived jobs and streams need explicit revalidation/cancellation at their application boundary. Do not promise retroactive cancellation by deleting a session row.

## Dependencies and prerequisites

Use the native `@better-auth/api-key` package matching the chosen Better Auth release, PostgreSQL persistence, existing HTTP infrastructure, and shared policy service. Reuse the plugin's key generation/hash/verification rather than inventing HMAC JWTs or a second API-key store.

The initial operator path uses migration/runtime configuration directly without opening public HTTP routes. The later administration path requires an active SuperAdmin with fresh MFA. PostgreSQL and both identity/native migrations must be available before provisioning.

## Database design

Native API-key hashes, native key IDs, permissions, reference IDs, expiry, and plugin counters live in `auth`. Never copy raw secrets or hashes into the identity schema.

| Service-owned table | Required fields and constraints |
| --- | --- |
| `identity.service_clients` | UUIDv7 ID, application ID, native service-user ID, name, allowed scopes, active/disabled status, version; each client belongs to exactly one app |
| `identity.service_credentials` | Native plugin key ID unique, client ID, active/revoked state, created/revoked times; no secret material |
| `identity.applications` | Approved public origin and enrollment policy; exact origin unique among active applications |
| `identity.admin_operations` | Provision/rotate/revoke operation ID, request fingerprint, status, non-secret progress, audit linkage |

Use logical native references and reconciliation rather than TypeORM-managed authentication entities. A native key created before a failed mapping write is unusable at ingress and is revoked by reconciliation. A mapping without a valid native key is equally unusable. Scope reduction and client disabling affect the next check even if native key revocation fails.

Native plugin permissions are a map of resource names to actions. Translate approved scopes explicitly: `auth:proxy` maps to resource `auth` action `proxy`; `sessions:validate` to `sessions/validate`; `identity:authorize` to `identity/authorize`. Do not accept an arbitrary permission map from a BFF or key owner. Key administration is SuperAdmin-only and cannot issue global human permissions.

## API contracts

All custom routes use phase 02 envelopes, validation, pagination, error shapes, and `Cache-Control: no-store`. Every internal request requires an active mapped service key and the named permission. A supplied `applicationId` or `x-app-id` must match the caller-bound application; it never overrides it.

### Backend contracts

| Method and proposed path | Request and caller | Response data | Errors and security effects |
| --- | --- | --- | --- |
| POST `/internal/v1/sessions/validate` | `sessions:validate`; body exactly one of `cookieHeader: string` (native auth cookies only) or `bearerToken: string` (signed native value without Bearer prefix), optional `requireFreshMfa: boolean` | `active: boolean`; when active, `user: UserView`, `sessionId`, `applicationId`, app `role`, `permissions: string[]`, `expiresAt`, `mfaVerifiedAt: timestamp|null` | 400 mixed/missing/malformed/oversized credential input; 401 service key invalid; 403 wrong app/scope/transport; 503 dependency error; audit suspicious mismatches, metrics for all outcomes |
| POST `/internal/v1/authorization/check` | `identity:authorize`; body exactly one of `cookieHeader` or `bearerToken`, `permissions: string[]` (max 20 registered names), optional `targetUserId`, `targetApplicationId`, `targetSessionId` | `allowed: boolean`, `reason: allowed|inactive_session|mfa_required|bootstrap_required|permission_denied`; return no user data on denial | 400 unknown permission/missing target/mixed credentials; 401/403 caller failure; 503 dependency error; audit privileged/suspicious denials |

For a valid service caller, an invalid/expired/stale/unassured human session returns 200 with `active:false` in validation and no identity/role fields. The authorization endpoint returns `allowed:false` and a safe reason. A membership mismatch is inactive. Backend/database failure returns 503, never an inactive/allowed success that hides an outage. Downstream consumers must deny on `active:false`, `allowed:false`, timeout, malformed responses, and non-2xx status.

This contract intentionally distinguishes session inactivity from caller authentication errors. The same human cookie submitted by another app's service credential never returns that user's identity. There is no endpoint accepting a user ID alone as proof of authentication.

The caller-bound application cannot be overridden: `targetApplicationId` describes an administration target, not a replacement caller identity. For own-account permissions, omitted target user means the authenticated user and a different target is denied. For session permissions, resolve any target session to that user before checking. Membership management requires target application/user; global user moderation/role permissions require target user, with the target-role restrictions from 08. Read-only capability queries may default to the caller's own application/user where unambiguous. Admin capabilities also require enrolled MFA, and mutations require the declared freshness. Missing/inaccessible targets yield a safe denial. A positive result is a point-in-time preflight, not a transferable grant: the actual mutation endpoint must repeat target, proposed-role, last-SuperAdmin, concurrency, and current-state checks. Downstream services must not turn it into authority over unrelated business resources.

### Service credential administration

| Method and proposed path | Request and caller | Response data | Side effects and errors |
| --- | --- | --- | --- |
| POST `/api/v1/admin/service-clients` | Fresh-MFA SuperAdmin; `applicationId`, `name`, `scopes[]`; `Idempotency-Key` | 201 client ID, application ID, name, allowed scopes, status, version | Create native service user and identity mapping via durable operation; 409 conflicting replay; audit `service_client.created` |
| GET `/api/v1/admin/service-clients` | Fresh-MFA SuperAdmin; app filter and pagination | Page of client metadata, no secrets | 401/403/503; audit privileged read |
| POST `/api/v1/admin/service-clients/:id/credentials` | Fresh-MFA SuperAdmin; `name`, `expiresInSeconds` up to 90 days; `If-Match` and `Idempotency-Key` | 201 key ID, `secret` once, expiry, client ID | Native key + mapping; audit without secret; 409 operation already used, 412/428 version precondition |
| DELETE `/api/v1/admin/service-clients/:id/credentials/:keyId` | Fresh-MFA SuperAdmin; `If-Match` | 204 | Disable mapping before native revocation; idempotent for already revoked owned key; audit `service_credential.revoked` |
| PATCH `/api/v1/admin/service-clients/:id` | Fresh-MFA SuperAdmin; `status` and/or reduced/approved `scopes`, `If-Match` | Updated metadata/version | Disabling immediately denies all keys; scope changes cannot exceed registered service scope catalog; audit changed metadata |

All write paths require durable operation intent. Secret-returning credential creation is deliberately not response-replayable: never save the secret in idempotency storage. If the response is lost after key creation, report the non-secret operation/key ID and require revocation and a new issuance. Retrying must not reveal an old secret or create uncontrolled extra keys. List metadata can include native key ID, name, expiry, status and last-use time where supplied; retrieval never returns a secret.

Default credentials expire after 30 days with an absolute configured maximum of 90 days. Rotation creates a second key, updates the BFF secret store, verifies traffic, then revokes the old key. The temporary overlap is explicitly tracked and normally limited to 24 hours.

### Initial operator provisioning contract

The proposed future command `identity:provision-application` is a single operator entrypoint, not an HTTP API. Inputs are application slug/name, exact HTTPS public origin, self-registration policy, client name, and fixed BFF/backend scopes. It creates/reuses the application, dedicated service user/client, and one initial native key under an exclusive operation lease. It writes progress/audit metadata, and outputs the new secret only to an interactive terminal or explicitly configured secret-manager sink. No logs, repo files, environment examples, or command-line secret arguments receive that secret.

Repeated execution with identical non-secret inputs returns the existing resource IDs without changing policy or retrieving/replacing credentials. Conflicting input fails. Recover partial native/identity writes through the saved operation; an unusable orphan key is revoked. Lost once-only output requires an explicit rotate operation. This bootstrap capability is available before HTTP starts and does not rely on a browser cookie, a hardcoded admin password, or the legacy plaintext `AUTH_INTERNAL_API_KEYS` map.

## Security considerations

Use TLS for backend calls and independent service secrets per application/client/environment. Keep keys in a secret manager; do not ship them to browsers, mobile bundles, source control, or request traces. Browser user sessions cannot call `/internal/v1/*` without a service key.

Native `/api-key/*` public HTTP paths are denied; provisioning invokes typed server-only APIs under in-process trusted operation context. Disable native session creation from keys. Bound metadata must be checked even when plugin verification reports `valid:true`.

Constrain session-cookie body size to 8 KiB and signed bearer values to 4 KiB, redacting both at every layer including failed validation. Accept exactly one allowed transport for the authenticated application. Do not forward unrelated application cookies or treat a bearer as a service key. Limit backend call time to two seconds initially, with a single request and no automatic mutation retry; deny when timed out. Metrics use route/method/outcome labels, not user IDs, cookies, keys, or raw origins.

The first release uses static key authentication over TLS. Mutual TLS and OAuth client-credentials grants are future enhancements; a service key must not be described as an OAuth access token or human refresh token.

## Implementation plan

1. During M1, configure the native key plugin, migrate its schema, and implement the operator application/BFF provisioning subset.
2. Enforce caller verification and exact app/host/forwarding binding before routing any native or custom application traffic.
3. Add the native route allowlist, cookie/header proxy contract, and handler-wide abuse controls.
4. After sessions/assurance/membership exist, implement validation and authorization with authoritative reads and safe denial results.
5. Add SuperAdmin-only client/key management, durable cross-pool operations, non-replayable secret issuance, and controlled rotation.
6. Replace the legacy plaintext internal-key environment mechanism through a documented deployment cutover; reject both absent and unrecognized mappings.
7. Verify two approved applications, a service-only caller, callback traffic, client/key rotation, storage failures, and downstream denial behavior before M5.

## Acceptance criteria

- INT-01: The first BFF can be provisioned without public unauthenticated management, and later identical runs do not reset/reveal secrets.
- INT-02: Every BFF key maps to exactly one active app/client; header/host/app spoofing and cross-app cookie reuse are denied.
- INT-03: Native hashing, expiry, scopes, and verification are reused; keys never create human sessions or confer global roles.
- INT-04: Proxies preserve bodies, callback status, native headers and multiple cookies, while denying internal/unknown routes.
- INT-05: Internal validation checks native state, current membership, epoch, MFA, and bootstrap on every call, with no success cache.
- INT-06: Timeout, unavailable stores, malformed replies, invalid credentials, and revoked sessions never produce allowed downstream access.
- INT-07: Key issuance/rotation/revocation survives partial failures without exposing stored secrets or leaving mapped active orphan credentials.
- INT-08: Standalone and microservice consumers integrate only over HTTP and need neither shared database access nor auth signing secrets.
- INT-09: Web cookies and native mobile signed bearer credentials resolve through the same authoritative app-bound policy, with no mixed-transport ambiguity or client-embedded service key; all MOB criteria in the mobile contract pass on iOS and Android.

## Official references

- [Better Auth API Key plugin](https://better-auth.com/docs/plugins/api-key)
- [v1.7.8 API Key options](https://github.com/better-auth/better-auth/blob/v1.7.8/packages/api-key/src/types.ts)
- [v1.7.8 API Key verification](https://github.com/better-auth/better-auth/blob/v1.7.8/packages/api-key/src/routes/verify-api-key.ts)
- [Better Auth dynamic base URL](https://better-auth.com/docs/reference/options)
- [Better Auth cookies and reverse proxies](https://better-auth.com/docs/concepts/cookies)
- [Better Auth sessions](https://better-auth.com/docs/concepts/session-management)
- [Nest guards](https://docs.nestjs.com/guards)
- [Express behind proxies](https://expressjs.com/en/guide/behind-proxies.html)
- [Selected Nest integration library](https://github.com/thallesp/nestjs-better-auth)
- [Required web and native mobile authentication](../14-mobile-authentication/README.md)
