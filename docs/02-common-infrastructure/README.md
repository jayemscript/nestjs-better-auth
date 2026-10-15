# Common Infrastructure

Release classification: first-release foundation, M1. Last reviewed: 2026-10-15.

[Documentation index](../README.md) · Prerequisite: [Dependencies](../01-dependencies-and-compatibility/README.md)

## Objective and scope

Make the existing application composition, configuration, and HTTP infrastructure safe to reuse. Extend the current common/database foundations instead of creating a second generic framework layer.

Both web and native iOS/Android authentication are required in the first release. Web/BFF requests use cookies; native resource requests use signed opaque bearer session credentials. The [mobile authentication contract](../14-mobile-authentication/README.md) defines the hosted system-browser sign-in, MFA, and verified-link handoff.

## Architecture and design

The root application composes validated configuration, the existing database module, Better Auth integration, identity policy, authorization, delivery, audit, and operations modules. The existing `DatabaseService` remains the TypeORM data-source boundary. Share a pure configuration parser between Nest startup and the standalone migration entrypoint; neither requires starting an HTTP server to load database settings.

Use the selected `@thallesp/nestjs-better-auth` 2.8.0 bridge through `AuthModule.forRootAsync`. Resolve the asynchronous native PostgreSQL/SSO-protected adapter before creating the single native auth instance; inject it through the bridge's `AuthService`. The bridge owns native handler registration; do not also mount `toNodeHandler` manually. Its `middleware(req, res, next)` option applies the shared authenticated-caller, route classification, and request-context policy before `next` enters the native handler. Native Better Auth hooks continue to protect effects and server API calls.

Set `disableGlobalAuthGuard: true` in the bridge and explicitly register a composite `APP_GUARD` for human Nest routes: invoke the library `AuthGuard` for native session resolution, then the authoritative service identity/application/assurance policy. Classify service-only/internal and health routes explicitly and enforce their appropriate service credentials or restricted health policy; anonymous metadata never exempts them from those checks. Bridge `@Roles` checks global native roles only and cannot replace application membership or MFA. `@Session()` can contain native credential material: map it into safe views and never return it directly. If bridge hook decorators are used, initialize native `hooks: {}` and `databaseHooks: {}` and register each semantic hook once.

Use one terminal Nest exception filter for custom routes. Rework the current error interceptor so it does not both send a response and rethrow. Align `CommonResponseDto`, transformation, Swagger, and error DTOs. Scope transformation to service-owned controllers; it must never alter native auth bodies, redirects, cookies, or 204 responses.

### HTTP composition order

1. Disable automatic body parsing when creating the Nest application.
2. Configure trusted proxies, request IDs, security headers, and exact origin/host and service-caller checks at the Express boundary. Separate the public, rate-limited mobile gateway from the private identity backend; only the gateway holds an application-bound service credential. Reject mixed cookie/bearer authentication on protected resource routes.
3. Apply streaming/request-size limits without consuming the native handler's body. Reject oversized requests at ingress and in the handler boundary; a `Content-Length` check alone is insufficient.
4. Let `AuthModule` register the single native handler at the configured `/api/auth` base path using its Express 5 routing support. Configure the bridge middleware policy before entry; reject the bare `/api/auth` path with 404 rather than redirecting unsafe methods. Confirm native routes bypass custom Nest guards/interceptors.
5. Keep Nest `bodyParser: false`; let the bridge's parser middleware skip the native auth base path before parsing custom APIs. Configure its JSON and URL-encoded limits to 64 KiB; no extra global parser may consume native streams. Native SAML bodies remain under their explicit 256 KiB ingress limit.
6. Register Nest validation, custom response transformation, the explicit composite guard and route classifications, exception filtering, and restricted Swagger.

Use a 64 KiB JSON limit for ordinary auth/custom requests and a 256 KiB route-specific limit for SAML ACS/metadata administration. Reject content types not required by that route. Streaming ingress limits apply even when chunked bodies omit a length.

## Dependencies and prerequisites

Use existing ConfigModule, class-validator/class-transformer, Swagger, and logger capabilities, plus the selected Nest auth bridge. The future dependency phase covers any additional security-header and health packages. Read phase 03 before finalizing database settings and phase 13 before finalizing service-key/bootstrap credentials.

### Configuration contract

All names below are planned configuration inputs; existing files remain unchanged. Keep current `PORT`, `NODE_ENV`, and `DB_*` connectivity names where practical.

| Group | Inputs and validation |
| --- | --- |
| Runtime | `PORT` integer 1..65535; `NODE_ENV` explicit development/test/production/maintenance; maintenance actually gates mutations |
| Shared DB | Existing host/port/name; verified TLS with `DB_SSL`, `DB_CA_CERT`; connection/statement timeouts and per-pool limits |
| DB identities | `AUTH_DB_USER/PASS`, `IDENTITY_DB_USER/PASS`, `AUDIT_DB_USER/PASS`; separate `MIGRATION_DB_USER/PASS` only in migration jobs |
| Schemas | `AUTH_DB_SCHEMA=auth`, `IDENTITY_DB_SCHEMA=identity`, `AUDIT_DB_SCHEMA=audit`; fixed validated identifiers, no request-selected schema |
| Auth | Versioned Better Auth secret/key ring, stable app name, exact `AUTH_PUBLIC_ORIGINS`, `AUTH_RP_ID`, bootstrap canonical origin |
| HTTP trust | Exact proxy topology/CIDRs, exact allowed hosts/origins, approved BFF mapping; do not treat hop count alone as caller authentication |
| Mobile gateway | Approved public native client/application mappings, exact hosted HTTPS authentication origins, verified HTTPS return links, iOS bundle IDs, Android package/signing fingerprints, transaction/HMAC key IDs, and bounded gateway attempt/expiry settings; no service secret in mobile bundles |
| SMTP | Host, port, TLS mode, username/password, from/reply-to address; TLS certificate verification mandatory in production |
| Delivery | `MAIL_ENCRYPTION_KEYS` with active key ID; worker concurrency/lease/retry settings |
| BFF bootstrap | One-time secret-manager input for initial service-key provisioning; replace `AUTH_INTERNAL_API_KEYS` plaintext map through a reviewed cutover |
| Providers | Google credentials; approved SSO secrets/metadata administered through restricted configuration paths |
| Operations | Swagger disabled by default in production; metrics access; retention and optional feature flags |

Reject missing secrets, malformed origins, duplicated app hosts, incompatible RP configuration, insecure production TLS, and invalid TTLs at startup. For multi-host auth, use the supported allowlisted dynamic base URL with HTTPS and no fallback for unknown HTTP hosts. Trusted proxy headers are enabled only behind the authenticated ingress that replaces them. Server-only calls without an incoming host must supply the approved canonical/request-specific context explicitly.

### Shared defaults

| Policy | Initial engineering default |
| --- | --- |
| Session | 7-day sliding expiry, refresh after 1 day, 5-minute native freshness |
| Cookie | Secure, HttpOnly session/state cookies; SameSite=Lax unless a specific native federation cookie requires a documented exception; host-only |
| Email verification link | 1 hour |
| Reset link | 15 minutes |
| Magic link and email OTP | 5 minutes; OTP six digits and at most five failed attempts |
| Service MFA completion | 5 minutes per pending session; no trusted-device bypass |
| Mobile handoff | 10-minute transaction, S256 PKCE, single-use native one-time-token code valid at most 1 minute; no session token in return URLs |
| Privileged assurance freshness | TOTP/recovery success in the previous 5 minutes |
| Email retry | At most four attempts including the first, each before challenge expiry |

These values are service policy choices, not claims about library defaults. Feature documents specify the exact enforcement owner. All timestamps use UTC/ISO 8601 with timezone; database timestamps use `timestamptz`.

## Database design

This phase has no new standalone entity. Request IDs are correlation metadata, not a new request table. Database, assurance, audit, and email schemas are defined in their feature documents. Reuse the existing base entity only for mutable TypeORM domain records with compatible lifecycle semantics.

## API contracts

### Custom API conventions

Success responses with a body use `statusCode: number`, `message: string`, `data: object|array`, `timestamp: ISO8601`, and `requestId: string`. An endpoint-specific table defines `data`. A 204 response has no envelope/body.

Errors use `statusCode`, stable service `code`, safe `message`, `timestamp`, `requestId`, and optional field-validation `details`. Exclude stack traces, raw database errors, token-bearing paths, and arbitrary exception objects. Preserve `Retry-After` on custom 429 responses. Native Better Auth errors retain their native shape and headers; clients must distinguish the route family.

| Status | Custom meaning |
| --- | --- |
| 400 | Malformed input or validation failure |
| 401 | Missing/invalid native session or service credential |
| 403 | Valid identity but denied app/permission/MFA/bootstrap policy |
| 404 | Missing or inaccessible resource without cross-app disclosure |
| 409 | Version/idempotency conflict or last-SuperAdmin invariant |
| 413 | Request exceeds route limit |
| 429 | Throttled with retry guidance |
| 503 | Required policy/storage dependency unavailable; not a successful authorization |

Use `limit` default 25/max 100 and opaque `cursor` on custom collection APIs. A page contains `items` and `nextCursor` (null when finished). Custom IDs are UUID strings; auth IDs are opaque native strings. Use a strong ETag derived from the custom record's version for mutable administrative resources; require `If-Match`, return 428 when absent and 412 when stale. Do not impose ETags on native auth APIs.

### Shared views

- UserView: `id`, `name`, `email`, `emailVerified`, `username?`, `image?`; no credential/provider tokens.
- SessionView: `id`, `createdAt`, `expiresAt`, `current`, `deviceLabel`, `lastSeenAt?`, `ipSummary?`, `mfaState`; never native session token.
- ApplicationView: `id`, `slug`, `name`, exact `publicOrigin`, `allowedTransports` (`cookie` and/or `bearer`), `status`, `version`; existing applications default to cookie only.
- MembershipView: `applicationId`, `userId`, `role`, `status`, `version`.

The native session result is not UserView/SessionView and can contain token material. Only the documented custom views are safe to return through general identity APIs.

No new endpoint is added by this phase. Audit/logging side effects apply to every custom endpoint: correlation and redacted operational completion; feature tables define additional security events.

## Security considerations

Validate DTOs with explicit whitelisted fields and reject unknown mutation fields. Derive actor IDs, application IDs, role, and assurance from authenticated server state. Do not use a client's `emailVerified`, `twoFactorEnabled`, `role`, `appId`, or user ID as authority.

Redact query strings and sensitive path segments before logging. Do not retain the current `originalUrl` behavior on reset/callback paths. Request IDs are generated or accepted only after format/length validation and are never used as authorization or idempotency keys.

Keep CORS and CSRF distinct. Require exact origins for browser mutations and same-origin BFF checks for custom cookie-authenticated routes; a preflight or successful CORS response is not authorization. Native protocol callbacks use their explicit protocol validation paths rather than a blanket origin bypass.

Apply the same browser rules to mobile's hosted authentication pages. Native bearer resource routes use a separately declared credential policy and never accept a cookie fallback; lack of a native-app Origin does not justify disabling global origin checks. Resolve both transports through typed Better Auth `getSession`, then apply identical current application, membership, lifecycle, epoch, and assurance checks. Mobile resume/foreground requires a fresh policy resolution; a cached profile is display data only.

## Implementation plan

1. Resolve missing root controller/service imports deliberately: remove unused scaffold references or replace them with the planned operations endpoints; do not duplicate application roots.
2. Extract one validated configuration contract and root initialization; reuse it in the existing database entrypoints.
3. Establish the module graph and provider lifecycle, including draining pools/workers on shutdown.
4. Initialize the bridge with the async native provider, pre-handler policy middleware, parser configuration, and disabled automatic global guard; set HTTP middleware order/size limits and preserve native response/header behavior.
5. Reconcile DTOs/interceptors and create the terminal filter and global custom validation behavior.
6. Register the explicit composite `APP_GUARD` and shared request context; classify human, service-only, and health routes deliberately, and retain native hook enforcement outside Nest guards.
7. Update future Swagger definitions to use cookies and service keys accurately; native routes get linked external contracts rather than fabricated Nest schemas.

## Acceptance criteria

- COM-01: Root composition resolves configuration/database providers without missing scaffold imports.
- COM-02: Runtime and migration settings use the same validated source; secrets are never logged.
- COM-03: Native auth receives an unconsumed stream and every native cookie/redirect/status is preserved.
- COM-04: Custom DTOs, responses, validation errors, and Swagger agree; errors are sent once and 204 is empty.
- COM-05: Chunked and declared oversized bodies are rejected; SAML has an explicit bounded exception.
- COM-06: Token-bearing URLs, headers, fields, and exception stacks do not appear in routine logs.
- COM-07: Unknown hosts, spoofed forwarding/application headers, and invalid production settings fail safely.
- COM-08: One bridge-managed handler receives the stream; its middleware establishes trusted context, explicit guards enforce human/service boundaries, and bridge sessions/roles/decorators never leak tokens or bypass application/MFA policy.

## Official references

- [Nest configuration](https://docs.nestjs.com/techniques/configuration)
- [Selected Nest integration repository](https://github.com/thallesp/nestjs-better-auth)
- [Nest validation](https://docs.nestjs.com/techniques/validation)
- [Nest exception filters](https://docs.nestjs.com/exception-filters)
- [Nest interceptors](https://docs.nestjs.com/interceptors)
- [Better Auth Express integration](https://better-auth.com/docs/integrations/express)
- [Better Auth options and dynamic base URL](https://better-auth.com/docs/reference/options)
- [Better Auth cookies](https://better-auth.com/docs/concepts/cookies)
