# SMTP Delivery

Release classification: first-release foundation, M1; required by credential recovery, verification, and passwordless milestones. Last reviewed: 2026-10-15.

[Documentation index](../README.md) · Prerequisites: [Persistence](../03-database-and-persistence/README.md), [Security](../04-security-and-audit-foundations/README.md)

## Objective and scope

Send authentication email through SMTP with service-owned plain-text and HTML content. Use Nodemailer 10.1.1 solely as the SMTP transport library. Do not add a provider SDK, hosted email API, Nest mailer module, email framework, rendering framework, or a second challenge generator.

Better Auth generates, hashes/stores where supported, verifies, expires, and consumes challenges. The service validates message context, encrypts a bounded delivery job, and dispatches it. SMTP acceptance is evidence that the relay accepted responsibility, not inbox delivery or proof that the recipient read the message.

## Architecture and design

```mermaid
sequenceDiagram
  participant BA as Better Auth callback
  participant Q as Delivery queue service
  participant DB as identity.email_deliveries
  participant W as In-process worker
  participant S as SMTP relay
  BA->>Q: Native URL or OTP + trusted application context
  Q->>Q: Validate origin, expiry, purpose; encrypt payload
  Q->>DB: Insert durable job or deduplicate
  Q-->>BA: Job accepted or enqueue failure
  W->>DB: Atomically claim eligible job with lease
  W->>W: Check expiry, decrypt, render owned content
  W->>S: Bounded TLS-protected SMTP send
  S-->>W: Accepted, rejected, or ambiguous timeout
  W->>DB: Mark terminal or schedule bounded retry
```

Callbacks await the durable enqueue result and perform no SMTP network call. Native APIs retain their configured enumeration-safe public response behavior; callback failures become generic native errors where propagated, with redacted operational evidence. Do not return a claim of delivery. Known-account queue insertion failure cannot be silently treated as successful durable delivery; expose the operational failure to native error handling and telemetry without disclosing whether a recipient exists.

The initial worker runs in the service deployment with a concurrency of two sends per instance and a pooled transport of at most two connections. Multiple replicas coordinate through database leases; no broker or separate worker service is required initially. New worker topology is an operational enhancement, not a prerequisite for this release.

### Failure guarantees

Native writes and the custom queue insert use independent pools. A callback is not a transactionally atomic outbox. Native challenge creation can commit without a job, or a job can exist for a native operation that subsequently fails. A consumed, superseded, or invalid native challenge may still be delivered; the native verifier must reject it. Do not attempt to restore consistency by independently generating a replacement token or extending its expiry.

Within the retry/expiry policy, an accepted job is dispatched with at-least-once attempt semantics. Worker crashes after relay acceptance can cause duplicate messages. A stable Message-ID aids diagnosis but does not guarantee relay/inbox deduplication. Bounded retries, permanent rejection, expiry, or a long outage can end in no successful send; there is no guaranteed delivery or exactly-once claim. Offer a rate-limited native resend flow with a fresh native challenge after recovery.

## Dependencies and prerequisites

Use the pinned Nodemailer transport, Node cryptography, the identity repository, and the existing scheduler/lifecycle facilities. Secrets come from configuration/secret management: SMTP credentials, sender address, TLS trust, active encryption key ID/key ring, and HMAC deduplication key. Configure domain SPF/DKIM/DMARC and approved sender policy with the actual relay/domain operator before launch; do not assume a transport library configures DNS.

### SMTP configuration

| Setting | Required policy |
| --- | --- |
| TLS | Port 587: `secure:false`, `requireTLS:true`; port 465: `secure:true`; reject any production mode allowing plaintext fallback |
| Certificate verification | `tls.rejectUnauthorized:true`, hostname verification, TLS minimum 1.2; custom CA for approved internal relay rather than disabling verification |
| Host | Explicit SMTP host; if using an IP, supply matching `tls.servername`; no user-supplied destination or transport URL |
| Authentication | SMTP username/password from secret manager; log neither credentials nor connection URLs |
| Timeouts | `connectionTimeout:10000`, `greetingTimeout:5000`, `dnsTimeout:5000`, `socketTimeout:10000` milliseconds; total per-send deadline 20 seconds |
| Pool | At most two connections per instance; worker concurrency two; account for total replicas and relay quota |
| Content security | `disableFileAccess:true`, `disableUrlAccess:true`; no attachments, arbitrary headers, remote content fetching, or user-selected sender |
| Logging | `logger:false`, `debug:false`, protocol transaction logging disabled; own allowlisted metrics only |

Explicit settings matter because transport defaults can be longer or less restrictive. [Nodemailer SMTP transport](https://nodemailer.com/smtp)

The total deadline must terminate the SMTP attempt rather than merely abandon a Promise while a socket continues sending. Use a bounded transport/connection lifecycle verified against the pinned transport. If the pool cannot safely abort one operation independently, close that worker-owned transport on deadline and recreate it after releasing affected claims; classify all potentially accepted sends as ambiguous. Prevent overlapping retry with a still-running SMTP transaction. Verify this behavior in later failure-injection work.

Transport `verify()` checks connection/authentication capability and can support deployment diagnostics, but it does not establish sender acceptance or inbox delivery. Do not send a real email as a readiness probe or block validation of existing sessions solely because SMTP is down.

## Database design

`identity.email_deliveries` uses no soft-delete base entity. It contains:

| Fields | Contract |
| --- | --- |
| `id uuid`, `created_at`, `updated_at` | UUIDv7 job identity and UTC timestamps |
| `purpose`, `template_version`, `application_id uuid` | Approved purpose (verify_email/reset_password/magic_link/email_otp/security_notice), owned content version, trusted app context |
| `payload_ciphertext bytea`, `payload_iv bytea`, `payload_tag bytea`, `payload_key_id text` | Nullable only in terminal states; encrypted recipient, native URL/OTP, safe display values, subject/body inputs, trusted return context |
| `deduplication_hmac text` UNIQUE | HMAC of canonical native challenge reference + purpose + app + recipient + template version, with key ID; no raw challenge or unkeyed short-OTP hash |
| `message_id text` UNIQUE | Stable opaque `<job-id@approved-sender-domain>` per job, reused on retry |
| `challenge_expires_at`, `not_before`, `next_attempt_at` | Native expiry supplied/derived from the exact configured callback contract; never later than native expiry |
| `state text` | queued/leased/retry_wait/accepted/failed/expired |
| `attempt_count smallint` | 0..4; increment when a worker claims an attempt, including crash/ambiguous attempts |
| `lease_owner text`, `lease_until timestamptz`, `version integer` | Claim/fencing token, 60-second lease, compare-and-swap version |
| `accepted_at?`, `terminal_at?`, `last_error_code?`, `last_smtp_class?` | Redacted metadata only; no relay free-text response, recipient, message body, or token |

Use a partial index `(next_attempt_at,created_at)` for queued/retry_wait rows, a partial index `lease_until` for leased rows, and indexes `challenge_expires_at`/`terminal_at` for cleanup. Keep recipient and native URL/OTP out of plaintext columns and index keys. Do not retain native tokens in audit records.

Encrypt payloads using AES-256-GCM with a fresh random 96-bit IV for every encryption and authenticated associated data containing job ID, purpose, app ID, and template version. Keep versioned keys outside the database; a database dump alone must not decrypt the payload. Retain decrypt-only old keys until all corresponding live jobs have expired or been processed; never drop a key while a live queued payload still requires it. Ciphertext authentication failure is terminal and alerts; no plaintext fallback.

### Claim and retry algorithm

1. Poll for due queued/retry_wait rows, using `FOR UPDATE SKIP LOCKED` and a short transaction. Expire rows whose challenge has less than 30 seconds remaining; do not claim them for send.
2. Set `state=leased`, unique owner token, `lease_until=now()+60 seconds`, increment attempt count/version, and commit before network I/O. Attempt limits and eligibility checks are part of the same transaction.
3. Recheck expiry and applicable application/account lifecycle, decrypt, render, and send with the stable Message-ID. The worker may use native-supported lookups for additional validity evidence, but expiry/lifecycle checks do not prove an unconsumed native token. Do not implement a second token verifier.
4. On relay acceptance for the one intended recipient, set accepted with time. On permanent recipient rejection (5xx), invalid content, or cryptographic failure, fail terminally. Connection/TLS/authentication outages and transient 4xx failures retry within the budget, with an alert for configuration failures. Timeout after DATA/acceptance uncertainty is ambiguous and retryable; duplicates are possible.
5. After attempts 1/2/3, schedule from failure time after 15/60/180 seconds respectively. Never schedule past challenge expiry minus the 30-second safety margin. Attempt 4 failure is terminal. No infinite automatic retry or token expiry extension.
6. Finalize with an update matching row ID, version, lease owner, and leased state. A stale worker cannot mark another worker's claim complete. Expired lease recovery counts the prior claim as an attempt and follows its retry budget; wait until lease expiry before reclaiming.

SMTP cannot enforce database fencing. A process paused for longer than its lease could resume after another claim. The worker must check current ownership before sending and enforce the bounded send lifetime; even then a crash/network pause can leave ambiguous acceptance. [PostgreSQL SELECT locking](https://www.postgresql.org/docs/current/sql-select.html) supports safe queue claiming, not exactly-once external side effects.

Purge ciphertext, IV, and tag in the same update that records accepted/failed/expired. A cleanup sweep expires abandoned rows and purges payloads within one minute of expiry; its outage is alertable. Retain only redacted terminal metadata for 30 days, then delete using the maintenance role. Deduplication lasts only while that metadata exists. Treat database/WAL/backups as secret-bearing until their own retention expires; row purge does not erase historical backups immediately.

## API contracts

This feature adds no standalone public `/send-email` endpoint. Email is issued only through native registration, verification, recovery, magic-link, and OTP operations documented in [Authentication](../06-authentication-and-account-lifecycle/README.md) and [Passwordless](../10-passwordless-authentication/README.md).

| Internal interface | Input / output | Failure and side effects |
| --- | --- | --- |
| `DeliveryQueue.enqueue` | Purpose, native recipient/URL or OTP, exact expiry, trusted application context, template version, request correlation → job ID and accepted/deduplicated | Validates approved URL origin and expiry, encrypts and persists; generic error on unavailable DB/key; audit `mail.queued` without recipient/token |
| `DeliveryWorker.dispatch` | Claimed job → accepted/retry_wait/failed/expired | SMTP and fenced row updates; events `mail.accepted`, `mail.retry_scheduled`, `mail.failed`, `mail.expired`; no claim of inbox delivery |
| Native email callback adapter | Native callback payload + server-bound context → completion/error | Enqueues only; does not return job IDs or queue internals to the browser |

Use the configured native challenge lifetimes from [Common infrastructure](../02-common-infrastructure/README.md): verification one hour, password reset 15 minutes, magic link/OTP five minutes. Obtain expiry from the native callback/configuration adapter rather than parsing an untrusted URL or defaulting to a fresh full lifetime after a delay. If a callback cannot provide/derive a sound bound, reject enqueue and fix the adapter before shipping that method.

Messages include the approved application name, purpose, expiry guidance, and safe support wording. Escape all HTML interpolation and separately encode URL attributes; include equivalent plain text. Validate link origins against the bound app. Never interpolate user input into headers/subjects or attach arbitrary files. Security notices do not contain a usable challenge; give them a bounded operational expiry of one hour and apply the same privacy/retry controls.

## Security considerations

Apply recipient/purpose/IP issuance throttles before native work through the shared policy, while preserving unknown-account response equivalence. Awaiting local durable enqueue still introduces timing differences between existing and unknown recipients; review native enumeration protections and measure status/body/latency distributions under success and queue-failure conditions. Equal response wording alone does not establish resistance to enumeration. Do not await SMTP in the request path. A resend cannot reset OTP/MFA guessing budgets. Do not send enrollment secrets or recovery-code lists by email. Email factors and verified email addresses do not satisfy the separately enrolled TOTP/recovery factor required by service MFA policy.

Disable click tracking and URL rewriting under service control for secret-bearing messages. Use no remote images in authentication messages. Rely on native one-time verification; email scanners can follow links, so clients must use the pinned library's documented flow and validate actual consumption behavior during implementation. Never invent a parallel safe-link verifier to compensate for provider behavior.

Limit SMTP egress to configured destinations. Alert on queue age, expiring jobs, repeated TLS/auth failures, ambiguous sends, retry exhaustion, payload purge lag, and key decryption failures. Keep telemetry labels low-cardinality and free of email addresses, job tokens, challenge IDs, and full SMTP responses.

## Ordered implementation steps

1. Add the reviewed transport dependency in future implementation and validate production TLS, credentials, sender, key ring, and timeouts at startup.
2. Implement owned escaped plain/HTML content and native callback adapters with exact expiry/application binding; prohibit generic arbitrary-email input.
3. Add encrypted queue records, deduplication, terminal purge, maintenance retention, and append-only audit events through the shared persistence boundary.
4. Implement atomic claims, fenced completion, bounded send cancellation, lease recovery, four-attempt expiry-aware retries, and graceful worker drain.
5. Connect every relevant native email callback; await durable enqueue, preserve native response/unknown-account behavior, and document multi-pool partial failures.
6. Add operational diagnostics/alerts and later verify TLS failure, ambiguous acceptance, duplicate dispatch, expiry, queue outage, rotation, and cleanup using controlled SMTP infrastructure.

## Measurable acceptance criteria

- MAIL-01: All challenge email uses the configured Nodemailer SMTP transport with owned content; dependency/import review finds no provider SDK or mail framework, and no independent challenge generator.
- MAIL-02: Missing STARTTLS, invalid certificates/hostnames, authentication failure, and stalled sockets fail within bounded time with no plaintext fallback or secret-bearing logs.
- MAIL-03: Queued payloads are authenticated/encrypted; raw recipients/links/OTPs do not appear in ordinary columns, logs, events, metrics, or retained terminal rows; key rotation preserves only still-needed decrypt capability.
- MAIL-04: Concurrent workers respect claims and fencing; attempt count never exceeds four; retries use 15/60/180-second delays and stop before expiry; crashed/ambiguous sends can duplicate but cannot claim exactly-once/inbox delivery.
- MAIL-05: Every native email callback awaits durable enqueue, validates app origin/expiry, preserves generic unknown-account behavior, and has documented orphan-job/missing-job recovery through native resend without extending a token.
- MAIL-06: HTML/header injection and hostile return URLs fail; terminal/expired payloads are purged within the stated window, metadata expires after 30 days, and an SMTP outage degrades email-dependent flows without authorizing a pending user or invalidating existing authorized sessions.

## Official references

- [Nodemailer SMTP transport, TLS, timeouts, and verify](https://nodemailer.com/smtp)
- [Nodemailer message configuration](https://nodemailer.com/message)
- [Better Auth email concepts](https://better-auth.com/docs/concepts/email)
- [Better Auth email/password](https://better-auth.com/docs/authentication/email-password)
- [Better Auth magic links](https://better-auth.com/docs/plugins/magic-link)
- [Better Auth email OTP](https://better-auth.com/docs/plugins/email-otp)
- [PostgreSQL SELECT locking](https://www.postgresql.org/docs/current/sql-select.html)
- [Node.js 22 authenticated encryption](https://nodejs.org/docs/latest-v22.x/api/crypto.html#class-cipheriv)
