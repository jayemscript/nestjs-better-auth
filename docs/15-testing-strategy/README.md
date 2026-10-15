# Testing Strategy and Acceptance Evidence

Release classification: first-release launch requirement, used throughout M0 through M5. Last reviewed: 2026-10-15.

[Documentation index](../README.md)

## Objective and scope

Define the future verification work needed to implement and release the service safely for web and native iOS/Android. Both mobile platforms are mandatory release gates. This document is a strategy and acceptance matrix; no application tests, builds, lint checks, migrations, provider callbacks, or SMTP tests were executed during documentation authoring.

Security controls are checked at the actual HTTP boundaries, not merely by invoking their helper functions. Existing scaffold tests and their `Hello World!` expectation are not evidence that authentication works.

## Architecture and design

Use the existing Jest/Nest testing foundation after its ESM compatibility work. Use real disposable PostgreSQL for schema grants, transactions, native plugins, concurrency, locking, and migration scenarios. In-memory substitutes cannot prove those properties. Use a controlled SMTP test relay that captures delivery only in a later authorized test environment, and separate real provider acceptance registrations from production credentials.

| Layer | Purpose | Required evidence |
| --- | --- | --- |
| Pure policy/unit | Configuration, permission mapping, redaction, time/expiry decisions | Deterministic cases including denied/default state |
| Persistence integration | Schema isolation, grants, atomic counters, job leases, cross-pool failure | Real PostgreSQL outcomes and controlled fault injection |
| Native contract | Pinned Better Auth paths, payloads, errors, cookies, hooks | Actual HTTP response contract snapshots with secrets normalized in memory |
| Full ingress HTTP | Native-handler and Nest guard policy, spoofing, BFF binding | Requests through production-equivalent middleware/proxy order |
| Browser/provider | Cookie semantics, Google, passkeys, enterprise OIDC/SAML | Real browser or supported virtual authenticator and sandbox IdPs |
| Recovery/operations | Restart, deployment compatibility, backups, rotations | Runbook results in an isolated environment |

No test infrastructure is added during this documentation task. In future implementation, place unavoidable caches, generated reports, logs, and package stores outside the repository or in an explicitly approved disposable workspace. Never create a repository-local `.pnpm-store`.

## Dependencies and prerequisites

Read each feature's acceptance IDs before implementing it. Future test environments need pinned runtime/packages, a disposable PostgreSQL instance matching production, dedicated non-production secrets, an SMTP capture relay, browser WebAuthn support, and sandbox Google/OIDC/SAML registrations. Use clock injection in service policy and controlled elapsed time; do not sleep for real challenge lifetimes where a deterministic clock can prove the policy.

Do not run provider-dependent cases against real users, production tenants, or externally addressed mail recipients. A sandbox lacking a required provider is a documented incomplete gate, not a passing release result.

## Database design

No production entity is introduced by testing. Test databases use the same `auth`, `identity`, and `audit` migrations and least-privilege roles. Seed fixtures through native APIs and legitimate operator provisioning rather than direct insertion of invented password/session rows.

Concurrency cases must use separate database connections and at least two service instances for shared state. Rollback-based unit fixtures alone cannot exercise native/TypeORM cross-pool boundaries. Record migration baseline/version, schema ownership, package versions, and fault timing in the external evidence record.

## API contracts

This phase adds no public endpoint. Test the contracts in 06 through 14 exactly, including native and custom error families and the separate mobile authentication task. Health/metrics are defined in 16.

For every exposed endpoint, include valid request, absent caller authentication, wrong application, wrong membership/permission, pending MFA, disabled/deleting account, invalid fields/content type, oversized/chunked body, dependency failure, and response redaction as applicable. For every native path not on the exposure allowlist, confirm the request cannot reach privileged functionality by method aliases, trailing slashes, path encoding, or alternate content types.

## Required scenario matrix

### Foundations and persistence

| ID | Future scenario | Required outcome | Acceptance linkage |
| --- | --- | --- | --- |
| V-01 | Missing production secret, invalid origin/RP, malformed TTL, insecure TLS config | Startup fails without echoing secret values | DEP, COM |
| V-02 | ESM development, compiled production, migration entrypoint, and test-runner imports | All use the reviewed module strategy and pinned packages | DEP-02 |
| V-03 | Fresh database and upgrade from each supported release | Correct objects in three schemas, migration ledger outside public, no automatic schema sync | DB-01 through DB-04 |
| V-04 | Native runtime role attempts identity/audit DDL or unauthorized reads | Database rejects operations; app has only declared grants | DB-05 |
| V-05 | Identity/audit roles attempt native credential access or event alteration | Access denied according to privilege matrix | DB-05, SEC |
| V-06 | Database certificate mismatch, expired CA, lost DB, exhausted pool | Verified TLS fails; readiness and API failure reflect outage; no auth bypass | DB, OPS |
| V-07 | TypeORM generated diff sees native/plugin tables | Native objects remain outside ORM entity diff ownership | DB-03 |
| V-08 | Migration process dies; second deploy races | One migration owner/lock, recoverable ledger, no HTTP auto-migration | DB, OPS |
| V-09 | Native write succeeds, independent identity/audit write fails | No false transaction guarantee; dependent access stays denied; operation reconciles | DB-07, ARCH-03 |

### HTTP security and mail

| ID | Future scenario | Required outcome | Acceptance linkage |
| --- | --- | --- | --- |
| V-10 | Native request stream after bootstrap and multiple Set-Cookie responses | Body intact, native status/location preserved, cookies separate | COM-03, INT-04 |
| V-11 | 204 or custom exception after interceptor/filter handling | Empty 204; exactly one safe error response | COM-04 |
| V-12 | Chunked oversized request, JSON confusion, large SAML assertion | Ordinary limit enforced; SAML bounded exception only | COM-05 |
| V-13 | Spoofed app ID, Host, X-Forwarded headers, direct backend request | Denied or trusted headers replaced; wrong app cannot be selected | COM-07, INT-02 |
| V-14 | Cross-origin mutation and legitimate OAuth/SAML callback | Malicious mutation denied; only validated protocol callback exceptions work | SEC, SSO |
| V-15 | Attack across two instances/IPs/challenges | Shared account and request budgets hold; temporary cooldown expires | SEC, MFA-07 |
| V-16 | Invalid/nonexistent email challenge request | Documented non-enumerating response; SMTP outcome does not reveal existence | AUTH, PWL |
| V-17 | SMTP unavailable, TLS downgrade, bad certificate, auth failure, 4xx/5xx | TLS enforced; retry classification, terminal state and alert correct | MAIL-01 through MAIL-04 |
| V-18 | Job expires before send or during retry backoff | No expired challenge submitted to SMTP | MAIL |
| V-19 | Two workers claim same job, lease expires, worker crashes after relay accepts | Bounded lease/concurrency; possible duplicate same challenge, no invented exactly-once claim | MAIL |
| V-20 | Queue/native transaction partial failure or response lost | Safe generic request failure/resend; no durable success claimed without job; orphan mail cannot authorize by itself | MAIL, AUTH |
| V-21 | Queue payload, logs, tracing, metrics, idempotency response inspected | No plaintext secrets; terminal ciphertext purged; labels bounded | MAIL-05, COM-06 |

### Accounts sessions and authorization

| ID | Future scenario | Required outcome | Acceptance linkage |
| --- | --- | --- | --- |
| V-22 | Registration, duplicate registration, username case/normalization, invalid credentials | Native contracts and uniform policy; no privilege fields accepted | AUTH-01 through AUTH-04 |
| V-23 | Unverified credential login and native verification replay | No normal credential login before verification; tampered/expired proof rejected; repeated valid signed email proof is idempotent and creates no session/membership | AUTH-03 |
| V-24 | Invalid/expired/replayed reset and two simultaneous redemptions | Native token rules enforced; old sessions revoked after successful reset | AUTH-04 through AUTH-05 |
| V-25 | Same-email OAuth account, unverified provider email, malicious linking, concurrent mixed-method removals | Explicit linking proof required; no silent identity merge; shared lock prevents removing every usable method | AUTH-07, EXT-08 |
| V-26 | Delete user while requests/providers/jobs remain active; restart between stages | Lifecycle denies access immediately; native/custom cleanup reconciles | AUTH-09 |
| V-27 | Session expiry/renewal boundary, remember-me false, app-bound cookie | Native behavior retained; host-only scoped cookies and binding enforced | SES-01 through SES-04 |
| V-28 | Revoke current/other/all sessions and check from another instance; native logout swallows DB deletion failure | Next authoritative lookup denies through invalidated assurance or native absence; no stale cache; admin/reset/current-user revocation use the correct native actor path | SES-05 through SES-08 |
| V-29 | Session list/revocation from another user or wrong application | No token disclosure or unauthorized session action | SES-04 through SES-05 |
| V-30 | Permission change and administrative restriction with existing sessions | Current policy/epoch wins; stale assurance never grants access | AZ-01 through AZ-05 |
| V-31 | App administrator mutates another app or grants global role | Denied; both HTTP and service method boundary checked | AZ |
| V-32 | Concurrent initial bootstrap and rerun with changed/existing email | One controlled bootstrap; no password reset or promotion by accidental collision | AZ-06 |
| V-33 | Bootstrap credential output lost and enrollment interrupted | No recoverable secret in logs/storage; explicit rotation/recovery; restricted account resumes legitimate setup | AZ-07 |
| V-34 | Two SuperAdmins simultaneously demote/ban/delete the other | At least one active SuperAdmin remains; operation serialization holds | AZ-05 |

### MFA and all sign-in methods

Run V-35 through V-42 separately for password, username, magic link, email OTP, Google, passkey, enterprise OIDC, and enterprise SAML wherever applicable. Test both library-mounted native HTTP routes and custom Nest routes. Repeat those method/factor cases through the hosted mobile browser flow on both iOS and Android; a successful web run does not count as mobile evidence.

| ID | Future scenario | Required outcome | Acceptance linkage |
| --- | --- | --- | --- |
| V-35 | MFA-enabled user completes only primary sign-in | No protected operation allowed; status/challenge flow only | MFA-02 through MFA-04 |
| V-36 | Pending user requests enrolled TOTP URI, code regeneration, passkey enrollment, email/password change | Denied even when the primary password is known | MFA-01, MFA-04 |
| V-37 | Native password challenge has null newSession; enrollment rotates session | No hook crash; only actual resulting session can become verified | MFA-05 |
| V-38 | Same accepted TOTP submitted concurrently from distinct sessions | At most one assurance grant; short-lived replay claim shared across instances | MFA-06 |
| V-39 | Same recovery code used concurrently or after successful use | One native winner; no duplicate assurance grant | MFA-06 |
| V-40 | Authenticated-native pending session brute-forces factor or alternates TOTP/recovery | Custom per-pending and per-account budgets hold despite native re-verification branch | MFA-07 |
| V-41 | Expired pending flow, changed epoch, switched app, factor mutation race | Fail closed; no assurance attached to wrong epoch/app/session | MFA-05, MFA-10 |
| V-42 | Identity outage after native factor success; trustDevice true; IdP amr claim | Restricted session remains denied; no bypass/substitute | MFA-08, MFA-10 |
| V-43 | SuperAdmin primary-only or missing enrollment attempts normal access/disable | Denied until setup complete; required MFA cannot be disabled | MFA-09 |
| V-44 | TOTP/recovery setup material and backup codes observed through logs/APM | No secret capture; no-store response; local QR generation | MFA-01 |

### Passwordless providers and backend integration

| ID | Future scenario | Required outcome | Acceptance linkage |
| --- | --- | --- | --- |
| V-45 | OTP purpose mismatch, failed-attempt limit, resend race, unknown user | Hashed native storage, correct purpose, bounded attempts; no automatic signup | PWL-01 through PWL-05 |
| V-46 | Magic-link scan/prefetch, expired/replayed link, callback to another app | Confirmation strategy prevents accidental GET redemption; native single use; app/origin binding | PWL-04 through PWL-05 |
| V-47 | Google callback state/code failure or wrong app callback registration | Rejected with safe error; no partial login accepted | EXT-01 through EXT-04 |
| V-48 | Passkey wrong RP/origin, absent user verification, replayed challenge, wrong owner deletion | Backend verified user-verification result and ownership gate reject | EXT-05 through EXT-09 |
| V-49 | OIDC wrong issuer/audience/nonce/state/sub, unverified email, SSRF discovery URL | Native verification and custom trust rules deny; no arbitrary role provisioning | SSO-02, SSO-06, SSO-08 |
| V-50 | SAML unsigned assertion, wrong audience/destination/recipient, expired/replayed InResponseTo, missing initiating-browser correlation | Strict protocol verification, shared replay store and transient browser binding; unsolicited IdP initiation rejected | SSO-03 through SSO-04 |
| V-51 | Provider configuration changed to attacker issuer/entity/certificate | Protected trust change denied or new-provider migration required | SSO-07 |
| V-52 | SSO secrets encrypted/decrypted through create/update/read/transaction/rotation | Native protocol operations work; missing key/corrupt ciphertext denies; no raw config leak | SSO-05 |
| V-53 | API key invalid/expired/unmapped/revoked/over-scoped; service account browser login | Caller denied; no human session conversion | INT-01 through INT-03 |
| V-54 | Internal validation with wrong app cookie, pending MFA, missing membership, disabled account | Inactive/denied result without user disclosure | INT-05 |
| V-55 | Internal validation DB outage, timeout, malformed response | 503 or failure; downstream denies; no positive cache | INT-06 |
| V-56 | Key creation mapping fails, once-only response lost, rotation overlap | Orphan key unusable; no stored secret replay; explicit revoke/reissue | INT-07 |
| V-57 | BFF path/proxy misuse and multiple applications in supported browsers | Internal/admin secrets hidden; cookies and callbacks remain app scoped | INT-04, INT-08 |

### Operations and release

| ID | Future scenario | Required outcome | Acceptance linkage |
| --- | --- | --- | --- |
| V-58 | Fresh deployment, compatible rolling deployment, incompatible native verification upgrade | Migrations precede traffic; incompatible changes use controlled drain/upgrade | OPS |
| V-59 | Backup restore into isolated environment with required secret key rings | Users/policies/events recover; sessions explicitly handled; no production emails sent | OPS |
| V-60 | Better Auth/SMTP/job encryption/service-key/SSO-certificate rotation | Planned overlap or forced sign-out; no silent undecryptable identities | OPS, SSO |
| V-61 | Shutdown mid-request/job and restart | Readiness withdrawn; bounded drain; lease recovery without unsafe permission grant | OPS |
| V-62 | Native/auth/identity/audit storage or external IdP/SMTP degrades | Health and alerts distinguish impact; safe policy and declared audit gaps | OPS, SEC |
| V-63 | Canary release and rollback after partial external side effect | Compatible rollback/runbook; no claim schema rollback reverses delivered email or consumed tokens | OPS |

### Required native mobile and Nest integration cases

| ID | Future scenario | Required outcome | Acceptance linkage |
| --- | --- | --- | --- |
| V-64 | iOS ASWebAuthenticationSession and Android Custom Tabs exercise each required method and factor | Registered HTTPS portal completes the same policy; no embedded WebView, missing method, or deferred-mobile release | MOB-01, MFA-11 |
| V-65 | Wrong/missing PKCE verifier, state, return URI, application, association file or browser binding | Exchange denied before native code consumption; no session in URL or another app | MOB-02 through MOB-03 |
| V-66 | Concurrent, expired, reused, intercepted or interrupted handoff; failed final response after native consume | At most one successful credential delivery; safe restart after partial failure; no secret-response replay | MOB-04, MOB-08 |
| V-67 | Pending MFA/bootstrap or disabled membership/provider at code generation or exchange | No bearer delivery or assurance bypass; existing native session cannot be upgraded by handoff | MOB-05 |
| V-68 | Signed bearer absent/tampered/unsigned, mixed with cookie, expired/revoked or copied across apps | Same authoritative policy as web; deny stale/ambiguous identity | MOB-06 through MOB-07, SES-09 |
| V-69 | App killed/resumed, device lost, cold-start verified link, offline logout, secure-storage inspection | No local/plaintext secret leakage or offline authorization; remote revocation works; offline server outcome reported accurately | MOB-08 |
| V-70 | Selected Nest bridge global guard disabled and explicit composite policy guard registered | Exactly one native handler/parser owner; Nest human routes use library session plus service policy, service-only/health paths use explicit classifiers; native routes remain protected by wrapper/hooks | COM-08, DEP-08 |
| V-71 | Bridge hook decorators/native hooks and native bearer after-hook ordering | Each semantic hook runs once; signed bearer captured only by guarded mobile exchange and stripped from browser responses | MOB-06, SEC-12 |

## Security considerations

Use synthetic users and controlled addresses. Never record live tokens or credential-bearing request/response snapshots in a repository or report. Redact secrets before evidence persistence, not after upload. Use ephemeral fixtures with deterministic assertions of no disclosure; do not print raw values as a debugging shortcut.

Verify the negative case and state transition, not only an HTTP status. A request returning 403 after issuing an application-authorized session or mutating a protected record is a failure. Likewise, a native operation that commits before custom failure must leave service access denied and record its recoverable partial state.

## Implementation plan

1. Establish the future ESM-compatible test/runtime baseline and approved isolated dependencies.
2. Implement meaningful policy tests alongside M1; add database and HTTP integration coverage before auth methods are enabled.
3. Add concurrency and fault-injection cases when each persistence boundary is introduced.
4. Maintain a native route/method exposure matrix and verify it against the pinned package on every upgrade.
5. Run the complete all-method MFA matrix and provider/browser acceptance during M4/M5.
6. Conduct restore, rotation, canary, and rollback drills in staging, then collect release evidence outside the repository.
7. Promote only when required feature acceptance and operational evidence are complete; record skipped/unavailable cases explicitly.

## Acceptance criteria

- TEST-01: Every feature acceptance ID has a verification scenario and a future evidence location.
- TEST-02: Native and Nest routes are covered through their real ingress paths, including deny/bypass cases.
- TEST-03: Real PostgreSQL proves grants, concurrency, native plugin behavior, and cross-pool failure recovery.
- TEST-04: All eight primary methods share the same enrolled-MFA matrix and pass independent web, iOS, Android, and provider checks; every MOB acceptance criterion has real-platform evidence.
- TEST-05: No required scenario is counted as passed through mocking, documentation review, or an unavailable environment.
- TEST-06: This documentation task executes no builds, lint checks, tests, migration tooling, generators, or provider probes.

## Official references

- [Nest testing](https://docs.nestjs.com/fundamentals/testing)
- [Better Auth testing utilities](https://better-auth.com/docs/plugins/test-utils)
- [Better Auth two-factor behavior](https://better-auth.com/docs/plugins/2fa)
- [Better Auth passkey testing and options](https://better-auth.com/docs/plugins/passkey)
- [PostgreSQL transaction isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- [Chrome virtual authenticators](https://developer.chrome.com/docs/devtools/webauthn)
