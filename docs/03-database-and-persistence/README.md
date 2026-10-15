# Database and Persistence

Release classification: first-release foundation, M1. Last reviewed: 2026-10-15. Status: implementation specification; migrations have not been executed.

[Documentation index](../README.md) · Prerequisites: [Compatibility](../01-dependencies-and-compatibility/README.md), [Common infrastructure](../02-common-infrastructure/README.md)

## Objective and scope

Persist shared authentication identities and service policy in PostgreSQL without placing application objects in `public`. Preserve the existing database module/data-source entrypoints while replacing their duplicated options, empty entity registrations, insecure TLS defaults, and public-schema assumptions in the future implementation.

Dedicated schemas organize ownership; grants enforce isolation. They do not isolate tenants or protect one module from another module running with all the same database credentials. This release has one shared identity realm.

The first release includes both web cookie and native iOS/Android signed bearer authentication. They use the same native users/sessions and service assurance records; mobile orchestration adds no parallel password, JWT, refresh-token, or session-credential store. See the [mobile contract](../14-mobile-authentication/README.md).

## Architecture and design

| Boundary | Writer and connection | Objects |
| --- | --- | --- |
| `auth` | Better Auth 1.7.8 supported PostgreSQL integration using its own `pg.Pool`, with narrow native SSO field protection | Native core, enabled plugin, and database rate-limit tables |
| `identity` | Existing Nest TypeORM data-source boundary | Applications, memberships, assurance, workflow orchestration, delivery, and service bindings |
| `audit` | Dedicated TypeORM data source using an insert-only runtime identity | Append-oriented security events |
| Migration ledger | Privileged TypeORM migration connection | `identity.schema_migrations`; one ordered history for all schemas |

Configure the native pool with connection startup `options: '-c search_path=auth'`. The baseline native integration accepts this pool directly. For the required SSO configuration protection, construct the standard native adapter from the same pool/options and apply only the narrow field decorator described below and in [Enterprise SSO](../12-enterprise-sso/README.md). Give every TypeORM entity an explicit `schema` and table name. Keep `pg_catalog` in PostgreSQL's normal implicit resolution; do not add a user-writable schema or `public` to any runtime search path. Schema names are fixed validated configuration, never request input. This pool form is supported by the [Better Auth PostgreSQL integration](https://better-auth.com/docs/adapters/postgresql).

The existing `DatabaseService` owns the identity data source. Separate providers own native and audit pools; inject abstractions rather than constructing pools per request. Use primary database reads for sessions, assurance, memberships, credentials, and revocation decisions. Do not introduce replica lag or successful authorization caching in release one.

### Migration ownership and transactions

TypeORM is the single deployment migration runner, with automatic synchronization and application-start migrations disabled. In a later implementation task, use the exact pinned Better Auth CLI/configuration to inspect generated SQL, review it, qualify every native object into `auth`, and incorporate the resulting statements into ordered TypeORM migrations. Do not run the native migration command against deployed environments alongside TypeORM. Capture the package/plugin versions and configuration fingerprint in migration review notes. Native generated SQL is an input to review, not authority to replace service tables.

Schema inspection has a separate entrypoint/profile built from the same validated full plugin/options configuration and raw native `pg.Pool` with the `auth` search path. Omit only the runtime SSO field-protection decorator/database factory: the native migration machinery needs its supported PostgreSQL configuration to generate SQL and cannot infer it from an arbitrary decorated adapter factory. Compare configuration fingerprints excluding that wiring difference so inspection and runtime enable exactly the same models/fields. This profile is for later reviewed schema inspection only; it is never a plaintext-production mode or a second deployment migration runner.

For mobile registration, `mobile_clients.auth_origin` equals its application's registered `public_origin`; use a distinct application registration for a mobile channel when its host differs from a web channel, and explicitly grant shared users the required memberships. Do not bypass the existing exact-origin checks by silently accepting another origin on the same row. The mobile transaction additionally stores an authenticated encrypted copy of client state only for its required return-link echo (`client_state_ciphertext`, IV, tag, key ID), alongside the comparison hash. Purge that encrypted echo at completion/expiry. No reusable native session secret or PKCE verifier is stored there.

The deployment migration job acquires a database advisory lock on one dedicated connection before checking/running the migration sequence. Concurrent jobs wait for a bounded interval and then fail deployment; they do not race. Use explicit transaction handling per migration; separate SQL that cannot run transactionally, and document forward repair. Apply reviewed grants/default privileges with the same object-owner role that creates the objects. A pool connection must not outlive its lock or transaction ownership.

Native and TypeORM pools cannot share a transaction merely because they use one database. A TypeORM `QueryRunner` covers only operations performed through that runner. A native user/session update, a delivery callback insert, and an audit insert can commit independently. Use durable orchestration records and fail-closed policy states for account deletion, bootstrap, role changes, credential rotation, and assurance promotion. Never describe callback enqueueing as an atomic transactional outbox with native writes. [TypeORM migrations](https://typeorm.io/docs/migrations/why/)

## Dependencies and prerequisites

Use the existing `pg`, TypeORM 1.1.2, Nest database module, UUIDv7 capability, and validated connection parser. Do not introduce a community Better Auth TypeORM adapter. Determine the real PostgreSQL version and connection allowance before deployment; PostgreSQL 18 is the proposed new-deployment baseline.

Use verified database TLS with a supplied CA when required. Initial per-instance pool maxima are native 5, identity 5, audit 2, plus one migration connection only during deployments. Limit rolling replicas so their aggregate maxima, maintenance connections, and database reserve fit the actual server limit. Start with connection acquisition 5 seconds, ordinary statement 5 seconds, lock wait 2 seconds, and idle transaction 10 seconds; migration jobs use reviewed operation-specific limits. These are engineering defaults requiring load evidence, not measured capacity.

## Database design

### Native objects and identifiers

Better Auth owns the exact native table/column names, defaults, indexes, constraints, and internal migrations. Generate the schema for the full enabled configuration: core user/account/session/verification, username, administration, two-factor, magic link, email OTP, passkey, SSO, API key, bearer, one-time token, and database rate limiting. Native one-time-token verification records remain in `auth`, with hashed token storage. A plugin need not create a separate table; its actual generated schema is authoritative. Never infer a second token/password table from the feature list.

Preserve native opaque string IDs and native expiry/deletion semantics. Service references to native users, sessions, providers, and API keys are `text`, not UUID. Do not map native objects as mutable TypeORM entities, extend the existing base entity, add soft deletion, or make native migrations depend on TypeORM metadata.

### Service-owned records

All timestamps use `timestamptz` in UTC. Custom entity IDs use UUIDv7. `version` is a positive integer incremented by compare-and-swap mutations; explicitly check affected-row counts. Reuse the existing base entity's ID/timestamp/version conventions only for applications and service clients where compatible. Exclude or deliberately override inherited soft-delete behavior: security policy cannot depend on an accidental implicit ORM filter. State machines and explicit lifecycle columns govern authorization.

| Qualified table | Essential columns and constraints | Required indexes and behavior |
| --- | --- | --- |
| `identity.applications` | UUID `id` PK; unique lowercase `slug`; `name`; exact unique `public_origin`; `allowed_transports text[]` default `['cookie']`, nonempty subset of cookie/bearer without duplicates; `status` active/disabled; `self_registration` boolean default false; `version`; timestamps | Unique slug/origin; disable stops new and existing application access; bearer transport requires active mobile registration |
| `identity.mobile_clients` | `application_id uuid` PK/FK; nullable `ios_bundle_id`, `android_package_name`; `android_signing_fingerprints text[]`; exact HTTPS `auth_origin`; `allowed_return_uris text[]` of exact verified HTTPS links; `status` active/disabled; `version`; timestamps | At least one configured platform; Android requires approved signing fingerprints; public registration conveys no user or backend authority; validate the hosted origin against the controlled web RP and enforce exact return links |
| `identity.mobile_auth_transactions` | UUIDv7 `id` PK; `application_id uuid` FK; exact `return_uri`; `platform` ios/android; S256 `code_challenge char(43)`; `client_state_hash`; nullable `browser_binding_hmac`, `native_session_id text`, `one_time_token_hmac`; `status` pending/browser_verified/code_issued/exchanging/completed/failed/expired; `expires_at`; nullable `code_expires_at`; `version`; timestamps | Index application/status/expiry and unique non-null token HMAC; ten-minute transaction and at most one-minute code; atomic state transitions and bounded expiry cleanup; no raw verifier, one-time token, bearer credential, or native session secret |
| `identity.application_memberships` | Composite PK `(application_id,user_id)`; app UUID FK; native `user_id text`; `role` member/app_admin; `status` active/suspended; `version`; timestamps | Index `(user_id,status,application_id)`; application delete restricted while memberships exist |
| `identity.account_policies` | Native `user_id text` PK; `security_epoch bigint` default 1; `bootstrap_state` password_change_required/mfa_enrollment_required/active; `kind` human/service; `lifecycle` active/deleting/disabled; `version`; timestamps | Positive epoch; native account lifecycle remains checked as well; missing row denies protected access |
| `identity.session_assurance` | Native `session_id text` PK; `user_id text`; `application_id uuid` FK; `security_epoch bigint`; `state` primary_only/verified; `primary_method`; nullable `mfa_method`, `mfa_verified_at`; `pending_expires_at`; creation/update times; `version`; optional native `provider_id`, provider-policy/application-mapping/identity-binding versions | Index `(user_id,security_epoch)`, `(user_id,application_id)`, provider and pending expiry; verified requires factor/time when account MFA policy requires it; one native session belongs to one application; provider-derived access rechecks current trust/binding state |
| `identity.auth_flow_bindings` | UUID ID; unique `flow_ref_hmac`; `purpose`; optional `user_id`; `application_id` FK; exact `public_origin`; `expires_at`; `created_at`; optional `otp_generation`, `browser_correlation_hmac`, `consumed_at`, native `provider_id` | Index expiry; HMAC of native flow reference with versioned service key; generation is a service sequence, never an OTP copy; browser correlation is keyed and atomically consumed; no duplicate OTP, state, PKCE, or link-token secret; binding expires with native flow |
| `identity.email_deliveries` | UUID ID; `purpose`; `template_version`; `application_id`; encrypted payload/ciphertext IV/tag/key ID; unique deduplication HMAC; unique stable `message_id`; expiry; state/attempt/lease fields | Full layout, claim query, expiry, and purge rules in [SMTP](../05-smtp-delivery/README.md) |
| `identity.service_clients` | UUID `id`; `application_id` FK; unique native `service_user_id`; `name`; `allowed_scopes text[]`; `status` active/disabled; `version`; timestamps | Index application/status; service account cannot sign in through human routes |
| `identity.service_credentials` | UUID ID; unique native `plugin_key_id text`; `service_client_id` FK; `state` active/revoked; timestamps | Index client/state; no credential-secret copy; native plugin verifies key validity and expiry |
| `identity.sso_provider_applications` | UUIDv7 `id`; unique `(provider_id text,application_id uuid)`; app FK; `enabled boolean`; `version`; timestamps | Index app/enabled; provider ID references native provider logically; membership and provider restrictions both apply; no redundant independently mutable status field |
| `identity.sso_provider_policies` | Native `provider_id text` PK; `status` provisioning/active/disabled/failed; `version`; timestamps; safe `last_operation_id` | Owns provider-view ETags and fail-closed lifecycle; non-active state denies use; native provider configuration remains native-owned |
| `identity.sso_identity_bindings` | UUIDv7 `id`; native `provider_id`; exact `issuer`, immutable `subject`; native `user_id`; `status` active/revoked; `approver`; `evidence_reference`; `version`; timestamps | Unique `(provider_id,issuer,subject)`; index user/status; immutable identity tuple and target user cannot be silently reassigned; no assertion or provider-token payload |
| `identity.admin_operations` | UUID ID; `kind`; `actor_id`; `application_id?`; unique actor/kind/idempotency-key HMAC; request fingerprint; target reference; phase pending/native_applied/policy_applied/completed/failed; safe result reference; timestamps | Index phase/updated time; no raw credentials/request bodies; reconciliation resumes known phases and checks native state |
| `identity.bootstrap_runs` | UUID ID; unique normalized bootstrap identity HMAC; native user reference nullable; phase; version; timestamps | Serialize bootstrap via advisory lock; retain no password or recovery code; record rotation/enrollment requirement |
| `identity.abuse_windows` | Composite PK `(scope,subject_hmac,window_start)`; count; `expires_at` | Atomic increment-and-check; bounded expiry index; short retention, no raw email/IP |
| `identity.mfa_replay_claims` | Composite PK `(user_id text,code_fingerprint text)`; `expires_at` | Atomic unique claim of keyed HMAC(user ID + accepted TOTP code), 90-second TTL; no raw TOTP; cleanup expired claims and deny reused live claims |
| `identity.operation_leases` | Resource/key PK; owner token; `lease_until`; version | For operations that need distributed serialization; renew/complete with matching owner, never treat expired ownership as authority |
| `audit.security_events` | UUIDv7 `id`; `occurred_at`; `request_id`; event/action; actor/user/application/target refs nullable; result; reason code; safe metadata; optional restricted IP/UA; correlation operation ID | Index `(occurred_at,id)`, `(user_id,occurred_at)`, `(application_id,occurred_at)` and operation ID; no soft-delete/version base entity; append-only runtime |

Use `CHECK` constraints for states and positive counters; size-limit text/JSON fields at DTO and storage boundaries. Grants do not substitute for input validation. Fixed owned JSON metadata fields are acceptable; do not persist arbitrary native responses.

Use physical FKs between service tables in `identity`. Native user/session/provider/key references are logical references checked through the native API and reconciliation, without cross-schema cascade FKs or TypeORM relations that mutate native data. This preserves native lifecycle operations and prevents audit history from blocking account deletion. Security decisions require the referenced native object to exist and pass current checks. Reconcile expired/orphan assurance/bindings daily and after account deletion; a stale service record never resurrects a deleted native identity.

Mobile handoff is a multi-pool workflow: validate complete session assurance and consume the proof-bound transaction before releasing a signed bearer credential. Native token consumption and identity transaction updates are not atomic across pools. Lost responses and uncertain consumption fail closed and restart authentication; retries must never mint a second credential from a completed, failed, expired, or in-flight exchange. Preserve the original native session ID, expiry, primary source, application binding, and factor assurance; one-time-token verification is a transport handoff, not a fresh primary sign-in.

SSO application mappings use `enabled` as their status; do not introduce a competing status column. Provider policy lifecycle is the separate `status` enum above. Subject approval records bind an exact cryptographically verified external issuer/subject to an already existing native user; they neither grant membership nor fabricate external proof. Revocation blocks existing provider-derived sessions as well as new sign-ins, through authoritative shared policy checks.

### Native SSO configuration protection

The pinned SSO provider's `oidcConfig` and `samlConfig` fields can contain client secrets and SP private keys; OAuth account-token encryption does not cover these serialized provider fields. Add the narrow authenticated envelope-encryption decorator specified in [12](../12-enterprise-sso/README.md) to the standard native PostgreSQL adapter. Construct it once during async Nest initialization through exported `getAdapter` using the full native configuration/pool, then return the prepared decorated adapter through the synchronous database factory. Encrypt those two fields on writes and decrypt all native read/result paths, including transaction-bound adapters and joins/lists. Bind ciphertext to model/field/native provider ID, preserve native transaction/predicate/locking/atomic-consumption behavior, and fail closed on unknown keys or corruption. Use secret-manager key resolution for SP private keys when supported. There is no TypeORM secret mirror or independently mutable credential copy; native indexes/resolution use unencrypted provider ID/issuer/domain fields. Round-trip, transaction, rotation and restore evidence is a launch gate.

Account deletion first commits `lifecycle=deleting`, increments the security epoch, and revokes access before invoking the native deletion API. Native self-deletion and administrative removal can have different hook behavior; both use explicit orchestration with an authenticated in-process operation context. Direct unwrapped native delete routes are denied. Cleanup removes custom personal state and pseudonymizes audit metadata, with reconciliation after partial failure. Native pending password/2FA flows can leave harmless unbound/expired rows; cleanup must never promote them to verified sessions.

### Privilege matrix

| Role | Grant | Explicit restriction |
| --- | --- | --- |
| Schema owner / migration job | Own reviewed objects; DDL and migration ledger in all three schemas; grant runtime access | Credentials unavailable to application/worker containers |
| Native runtime | CONNECT database, USAGE `auth`, required CRUD and sequence access on native objects | No CREATE; no access to `identity`/`audit`; no ownership; not a migration runner |
| Identity runtime | USAGE `identity`; required CRUD/sequence privileges | No native or audit access; no DDL; revoke unneeded table operations where practical |
| Audit runtime | USAGE `audit`; INSERT on events only | No SELECT/UPDATE/DELETE/TRUNCATE; RETURNING must not require unavailable SELECT privileges |
| Audit reader | SELECT on restricted audit views/tables | No mutation; available only to authorized operational tooling |
| Retention maintenance | Narrow DELETE/field-purge privileges for approved retention jobs | Separate credentials/job; no schema ownership |

Revoke CREATE on `public` and unnecessary PUBLIC schema access in the dedicated service database; no service object lives there. Review grants inherited through role membership and default privileges for both current and future tables/sequences. Schema isolation relies on those permissions, as described by [PostgreSQL schemas](https://www.postgresql.org/docs/current/ddl-schemas.html). A separate audit connection reduces accidental modification; a compromised process holding every runtime credential can still submit fabricated audit entries. Stronger external immutable audit storage is a later enhancement.

## API contracts

This phase creates no HTTP endpoint. Internal persistence interfaces are `NativeAuth` (typed supported native server APIs), `IdentityRepository` (policy reads and transaction-scoped custom mutations), `AuditWriter.append(event)` (durable success or error), and `DeliveryQueue.enqueue(message)` (durable job ID or error). Their inputs carry authenticated application/request context; they never derive permissions from a caller-selected schema.

Migration jobs are deployment operations, not public APIs. Health APIs report dependency readiness without exposing connection strings, schema inventories, migration SQL, or raw errors. Administrative resource methods and concurrency contracts are specified in [Authorization](../08-authorization-and-bootstrap/README.md).

## Security considerations

Treat password hashes, TOTP material, provider refresh/access tokens, verification records, session tokens, API-key metadata, and encrypted queued mail as sensitive. Restrict database backups and restore environments to the same confidentiality standard. Retain native library encryption/hashing contracts; SMTP payload encryption is additional protection for deliverable secrets, not a replacement for native challenge hashing.

Parameterized queries and schema-qualified identifiers are mandatory. Do not interpolate user identifiers or migration SQL fragments. Readiness fails if required schemas/version/grants cannot support policy decisions. Graceful shutdown stops claims and new requests, drains bounded in-flight work, then closes the native pool and both TypeORM data sources; terminate any abandoned transaction safely.

## Ordered implementation steps

1. Refactor the existing runtime and CLI database options to one validated configuration source; preserve future ESM compatibility.
2. Provision schema owner/runtime/reader/maintenance roles and verified TLS; establish pool budgets and startup search paths.
   Implement the minimal server-only application/service-client provisioning and authenticated ingress subset from [Integration](../13-application-and-service-integration/README.md) in M1 so the HTTP trust boundary can be configured without a bootstrap dependency cycle; the complete integration milestone remains M5.
3. Enumerate the full pinned native plugin schema; review SQL and embed it with service DDL/grants in the single ordered migration sequence.
4. Register only service-owned TypeORM entities explicitly; keep native authentication tables outside synchronization and metadata diff generation.
5. Implement repositories, compare-and-swap updates, claim/lease transactions, and logical-reference validation.
6. Implement durable administration/bootstrap reconciliation and retention/orphan cleanup with independent privileged jobs.
7. Implement readiness/shutdown behavior; collect future migration/grant/concurrency evidence from the [Testing strategy](../15-testing-strategy/README.md) before promoting M1.

## Measurable acceptance criteria

- DB-01: A fresh migration creates every required native/plugin and service object only in `auth`, `identity`, or `audit`; same-named decoy objects in `public` do not affect queries.
- DB-02: Native, identity, and audit connections pass an explicit privilege matrix; runtime DDL/cross-schema access and audit modification attempts fail.
- DB-03: There is one migration history and one deployment runner; two concurrent jobs cannot apply the same migration; failed migration recovery is documented and rehearsed later.
- DB-04: Native IDs remain strings and native lifecycle remains unchanged; custom mutable records use explicit UUIDv7/version conventions without imposing soft deletion on native/auth/audit records.
- DB-05: Concurrent assurance promotions, membership edits, queue claims, and bootstrap calls preserve state/version invariants and produce no duplicate active workflow effect.
- DB-06: Failure between native and custom commits yields a durable recoverable phase or denied policy state; reconciliation cannot authorize an orphan user/session.
- DB-07: All authorization reads use authoritative primary state; unavailable pools or missing policy rows never produce an authorized result.
- DB-08: Pool accounting, verified TLS, transaction timeouts, migration rollback/forward repair, restore order, and graceful shutdown have future acceptance evidence; no runtime validation is claimed by this document.
- DB-09: Mobile configuration and transaction records are schema-qualified in `identity`, contain no recoverable session/token/verifier copy, and enforce single-use state transitions; native one-time-token records stay in `auth` and concurrent/partially committed exchanges never authorize twice.

## Official references

- [Better Auth PostgreSQL integration and schema selection](https://better-auth.com/docs/adapters/postgresql)
- [Better Auth database concepts](https://better-auth.com/docs/concepts/database)
- [TypeORM schemas and multiple data sources](https://typeorm.io/docs/data-source/multiple-data-sources/)
- [TypeORM migrations](https://typeorm.io/docs/migrations/why/)
- [PostgreSQL schema privileges](https://www.postgresql.org/docs/current/ddl-schemas.html)
- [PostgreSQL SELECT locking clauses](https://www.postgresql.org/docs/current/sql-select.html)
