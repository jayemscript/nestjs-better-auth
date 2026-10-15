# Deployment and Operations

Release classification: first-release production launch gate, M5. Last reviewed: 2026-10-15.

[Documentation index](../README.md) · Prerequisites: all first-release feature contracts and [verification strategy](../15-testing-strategy/README.md)

## Objective and scope

Specify a deployable, recoverable operating model for the service. Reuse the existing container approach, PostgreSQL, and SMTP. A managed container service or ordinary VM/container host is sufficient; Kubernetes, Redis, a broker, and multiple regions are not required by this design.

These are runbook specifications for future implementation. No deployment, container build, migration, health probe, backup, restore, load test, or external side effect is performed in this documentation task.

## Architecture and design

### Deployable components

Initially run the API and bounded email dispatcher in one service process with shared PostgreSQL leasing. It can scale to multiple replicas without in-memory security state or duplicate active job claims. A separate worker process using the same delivery contract is a later production enhancement, not a prerequisite. Size all database pools across maximum replicas, workers, migration jobs, audit reads, and operator reserve.

The current Dockerfile is a starting point. Future work pins a supported Node 22 patch at least 22.22.1 for the selected Nest integration library, its image digest, and the pnpm version; reviews lifecycle scripts; builds in a builder stage; and copies only production essentials. Run as an unprivileged user with read-only application files and no unnecessary upload directory. Keep package stores outside the checkout/build context and secrets out of layers. The production image contains compiled migration code for the dedicated migration job or a separate reviewed migration image.

The private backend accepts application traffic only from authenticated registered BFF/gateway/backend clients. Public mobile clients reach only their mobile gateway, which holds the backend key. Publish provider callbacks through approved web or mobile-hosted authentication origins; do not expose internal validation/metrics to browsers or devices. Use TLS between components where traffic crosses trust boundaries and verify database/SMTP certificates.

First-release mobile deployment additionally requires exact registered HTTPS authentication/return hosts, iOS associated-domain/AASA and Android Digital Asset Links ownership with package/signing fingerprints, hosted system-browser authentication pages, encrypted short-lived transaction state, shared gateway abuse/lease storage, and Keychain/Keystore client storage. Deploy and verify these integration components before promotion; mobile is not an optional post-launch item. Keep raw native one-time-token HTTP endpoints denied. Monitor expired/failed exchanges and association-file drift without logging codes, verifiers, states or bearer credentials.

### Startup and shutdown

Startup validates configuration, initializes bounded auth/identity/audit pools, verifies expected migration/schema version without mutating it, creates the native integration and policy hooks, and then marks readiness. Startup never creates tables, runs migrations, generates credentials, or seeds an admin implicitly.

On termination, withdraw readiness immediately, stop claiming new mail jobs, drain active HTTP requests and SMTP submissions for at most 30 seconds, then close transport/pools. A send is bounded below the delivery lease lifetime. Unfinished jobs recover through their lease; SMTP may already have accepted a message, so the next attempt can duplicate delivery.

## Dependencies and prerequisites

Actual deployment inputs are: controlled application origins/common WebAuthn RP, PostgreSQL server/version/CA, SMTP relay and verified sender domain, secret-manager locations, Google callbacks, OIDC/SAML trust registrations, approved ingress paths, telemetry destination, on-call owner, and measured capacity. Missing values are deployment blockers; examples are not usable production credentials or hosts.

Before release, obtain sender-domain SPF/DKIM/DMARC configuration from the mail operator, relay quotas/bounce procedures, and test recipients. SMTP acceptance is not delivery assurance; the service has no universal provider SDK webhook integration. Monitoring of relay acceptance and operator-managed bounce/delivery feedback must be described accurately.

### Release manifest

Record image digest, exact application/Better Auth/plugin/CLI versions, configuration fingerprint without secrets, migration range, enabled methods, public origins, RP ID, native route exposure version, secret key IDs, and rollback compatibility. Security policy changes are versioned and reviewed separately from arbitrary deployment environment strings.

## Database design

One privileged migration runner owns changes across `auth`, `identity`, and `audit`, including the migration ledger. Runtime roles have no DDL. Use the reviewed TypeORM migration sequence with imported native SQL; never launch the Better Auth migration CLI independently against production.

### Backup and restore

- Back up all three schemas and their migration ledger as one PostgreSQL recovery unit. Enable point-in-time recovery when supported by the chosen platform.
- Store encryption/secret key rings independently with controlled backup and restore access. A database backup alone cannot decrypt TOTP, provider tokens/configuration, or queued mail.
- Proposed initial objectives: RPO at most 15 minutes and RTO at most 4 hours. These are engineering targets requiring platform validation and a restore drill before launch.
- Restore first into an isolated environment with SMTP dispatch disabled and external callbacks blocked. Validate schema versions, record integrity, grants, key availability, and account policies before any production cutover.
- Treat native sessions/challenges in restored backups as potentially resurrected. During a real security/disaster restore, force session/challenge invalidation and increment service security epochs before opening traffic; rotate secrets deliberately where needed. Do not rely on restored expiry alone.
- Do not replay old delivery jobs automatically after restore. Expire stale challenge deliveries and require fresh user requests.

### Retention and deletion

Use the default policy in phase 04: security-event metadata 90 days, full IP/user-agent access restricted and minimized to 30 days, email metadata 30 days, and immediate removal of encrypted secret payload on terminal delivery. Operational logs initially retain 14 days. Backups initially retain 30 days with documented deletion lag; do not claim immediate removal from immutable backup history.

Run maintenance with a dedicated role that can expire records under policy. Runtime audit writers cannot update/delete evidence. Deletion reconciles native and service-owned records and pseudonymizes retained audit data; supplied jurisdiction/contract requirements may revise these defaults before launch.

## API contracts

Operational endpoints are service-owned, excluded from ordinary browser proxy routes, and do not use the general identity success envelope. Keep the surface small and non-secret.

| Method and proposed path | Caller/request | Response | Errors, side effects, and evidence |
| --- | --- | --- | --- |
| GET `/health/live` | Local/load-balancer network; no body | 200 `{status: ok}` while event loop/process responds | No DB/SMTP call; no per-probe security event; bounded metrics |
| GET `/health/ready` | Private orchestrator/ingress probe | 200 `{status: ready}` only when config/schema/auth/identity pools and required policy dependencies are usable; 503 `{status: not_ready}` otherwise | Public response omits credentials/topology; state changes logged once and alertable |
| GET `/metrics` | Dedicated monitoring network and configured monitoring credential | Prometheus-compatible text; content type declares format | 401/403 outside authorized monitoring; no user/key/IP labels; audit configuration access, not every scrape |

Detailed internal health can report named dependency statuses only to authorized operators. An SMTP outage does not make existing session validation unready. It marks email authentication degraded through telemetry and launch policy. Required audit-intent storage failure disables affected privileged mutations; if the deployment cannot safely serve any declared required policy path, readiness is withdrawn. Never make liveness depend on a downstream outage and cause restart storms.

### Signals and initial targets

| Signal | Proposed threshold or objective | Action |
| --- | --- | --- |
| Internal validation availability | 99.9% successful service handling per calendar month, excluding invalid-session/credential decisions | Investigate budget burn; stop promotion on regression |
| Validation latency | p95 below 200 ms at declared healthy-dependency workload | Measure in staging; tune capacity/pools before promise |
| 5xx/503 rate | Above 1% for 5 minutes, at least 100 requests | Page configured owner; inspect store/pool/deploy state |
| Auth failure pattern | At least 20 failed primary attempts per minute for 5 minutes with distributed/account anomaly signals | Security alert; do not page on each invalid password |
| MFA replay/bypass detection | Any accepted forbidden-path attempt or confirmed policy bypass | Treat as incident and disable affected method |
| Mail delay | Oldest eligible pending job above 60 seconds for 5 minutes, or terminal failures above 5% of at least 20 submissions | Alert mail operator; protect resend budgets |
| Database pool | Above 80% allocation for 5 minutes or timeout count rises | Reduce concurrency/scale carefully; avoid aggregate connection exhaustion |
| Audit/reconciliation | Any privileged intent cannot persist; incomplete operation older than 5 minutes | Deny unsafe mutation, alert and reconcile |
| Backup/recovery | Failed backup, recovery lag above 15 minutes, or missing key backup | Block launch/promotion and repair |

Thresholds are initial engineering defaults, not production measurements. Record the actual workload and observation windows. Keep provider outages, user denials, rate-limit denials, and application errors separate. Never label metrics with raw usernames, emails, session IDs, API keys, tokens, full IPs, arbitrary provider URLs, or unconstrained request IDs.

## Security considerations

### Rotation procedures

| Material | Procedure |
| --- | --- |
| Better Auth secret/key ring | Use versioned-secret support verified for the pinned version; preserve old decrypt/verify keys for live stored data, migrate deliberately, and test native state/cookie handling. For suspected compromise, force session/challenge invalidation rather than preserving trust blindly |
| Mail payload encryption | Add new key ID, write with it, retain old keys until live queued payloads expire/decrypt; then retire under evidence |
| SSO provider configuration encryption | Rotate the field-protection key ring through read/decrypt/re-encrypt compatibility; corrupt/missing-key config denies sign-in; never silently replace with plaintext |
| Service/BFF key | Issue overlap key, deliver through secret manager, verify new traffic, revoke old mapping/native key within tracked window |
| Google/OIDC client secret | Use provider-supported overlap/rotation; keep issuer/client identity unchanged; verify callbacks in staging |
| SAML certificates | Explicit authorized overlap for same IdP authority, shared replay/correlation state, then retire old certificate after provider cutover |
| Database/SMTP credentials or CA | Stage compatible trust/credential changes, reconnect pools/transports, verify before removing old access |

Do not conflate cookie secret rotation with re-encrypting persistent TOTP/provider data. Preserve needed old keys or explicitly invalidate dependent data. Never print keys during an operational probe or write them to repository-local files.

### Incident procedures

- Suspected credential/session compromise: disable affected account/client/provider, increment relevant security epochs, revoke native sessions/keys, preserve audit evidence, rotate compromised material, and require fresh proof.
- Method-specific bypass: disable that sign-in method/callback exposure, deny affected pending sessions, and keep other methods only if their shared policy remains trustworthy. Do not relax MFA to restore availability.
- SMTP failure: retain bounded jobs, stop expired delivery, communicate safe resend behavior, and restore relay access; do not switch to an unapproved email API.
- PostgreSQL failure: deny authentication/authorization, withdraw readiness where required, preserve retry budgets, and avoid optimistic authorization from stale client state.
- Partial administrative work: inspect durable operation ID, compare native and identity state, finish/revert through the documented idempotent action, and append the actual outcome. Never mark success solely because the request previously returned 202.

## Implementation plan

1. Provision the three schemas/roles, managed secrets, private network, SMTP relay/sender, and monitoring destinations.
2. Harden/pin the container and tooling; remove unnecessary runtime write paths and broadly permitted install scripts during later implementation.
3. Apply reviewed migrations once with an exclusive deployment lock before starting compatible application instances.
4. Provision the initial application/BFF credential through the operator contract in 13, then bootstrap SuperAdmin through 08 and complete legitimate verification/setup.
5. Configure each application's Google/SSO callbacks, native host/origin/RP policy, route allowlist, and all-method MFA hooks. Confirm all first-release methods in staging.
6. Implement restricted health/metrics, maintenance/drain behavior, retention, and reconciliation. Run verification and restore/rotation drills only in the later implementation environment.
7. Require staging evidence for web and real iOS/Android authentication, all-method MFA, verified return links, concurrent/expired handoff, secure storage and cross-platform revocation. Then release to a limited web and mobile audience for at least 24 hours, observing the defined targets and no unresolved security-policy failures; expand to remaining approved applications only after those gates pass.
8. Roll back application images only when the previous image is compatible with current migrations and native verification formats. Use forward repair when not compatible; do not blindly reverse destructive migrations.

### Upgrade and rollback constraints

Use additive expand/contract migrations and retain old readers/writers during rolling compatibility windows. Authentication libraries may change verification identifiers, cookie state, or plugin schema semantics. Review pinned release notes/source; some upgrades require coordinated instance replacement and restarting pending flows rather than mixing versions. Native schema migration downgrade does not reverse already consumed tokens, emitted email, provider registrations, or deleted accounts.

The rollout record must name a responsible operator and rollback deadline. No real owner was supplied here; assigning the deployment owner and notification destination is a mandatory launch input, not an invented team commitment.

## Acceptance criteria

- OPS-01: Images/tooling are pinned, runtime is unprivileged, secrets are external, and no package store/artifacts are created in the repository.
- OPS-02: One migration owner applies reviewed changes; runtime only checks compatibility and cannot perform DDL.
- OPS-03: Health distinguishes process liveness, policy readiness, and email/provider degradation without leaking topology/secrets.
- OPS-04: Defined alerts have a real owner/destination, bounded labels, and verified failure signals before launch.
- OPS-05: An isolated restore proves the proposed RPO/RTO and required key recovery, with mail disabled and restored sessions invalidated deliberately.
- OPS-06: Key/credential/certificate rotation procedures preserve explicit compatibility or force safe re-authentication; no silent plaintext fallback.
- OPS-07: Graceful shutdown, job recovery, partial-operation reconciliation, and compatible rollback are demonstrated in the future staging environment.
- OPS-08: Every README method and mandatory security gate passes before full release; canary expansion follows at least 24 hours of acceptable observed operation.
- OPS-09: Both iOS and Android complete the mobile contract before release; link ownership, public gateway/private backend boundaries, encrypted transaction cleanup, bearer redaction and device-session revocation are deployed and monitored.

## Official references

- [Nest health checks](https://docs.nestjs.com/recipes/terminus)
- [Nest application lifecycle and shutdown](https://docs.nestjs.com/fundamentals/lifecycle-events)
- [Nest logging](https://docs.nestjs.com/techniques/logger)
- [PostgreSQL backup and restore](https://www.postgresql.org/docs/current/backup.html)
- [PostgreSQL continuous archiving and PITR](https://www.postgresql.org/docs/current/continuous-archiving.html)
- [Better Auth options and versioned secrets](https://better-auth.com/docs/reference/options)
- [Better Auth database migration concepts](https://better-auth.com/docs/concepts/database)
- [Better Auth session revocation limitations](https://better-auth.com/docs/concepts/session-management)
- [Nodemailer SMTP transport](https://nodemailer.com/smtp)
