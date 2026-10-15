# Google Authentication and Passkeys

Release classification: first release, M4. Status: proposed; no provider integration has been deployed. Last reviewed: 2026-10-15.

[Documentation index](../README.md) · Prerequisites: [lifecycle](../06-authentication-and-account-lifecycle/README.md), [sessions](../07-session-management/README.md), [authorization](../08-authorization-and-bootstrap/README.md), [MFA](../09-multi-factor-authentication/README.md)

## Objective and scope

Support Google authorization-code sign-in and discoverable WebAuthn passkeys for the same shared identities. Provide passkey enrollment, listing, naming, deletion, and recovery through other enrolled methods. Google and passkey sign-in must obey the same membership, account-state, bootstrap, and MFA policy as password authentication.

Web and native iOS/Android are required first-release clients. Mobile opens its approved HTTPS authentication portal in the platform system browser; Google callbacks and WebAuthn ceremonies finish there using the existing exact origins/RP, then local MFA completes before the [mobile one-time handoff](../14-mobile-authentication/README.md). This requires no embedded WebView, native Google ID-token path, Expo dependency, or separate mobile passkey store. Device-specific native provider/passkey UI is optional later; both mobile platforms must pass the hosted-browser methods now.

## Architecture and design

### Google

Use Better Auth's built-in Google provider. Request only `openid`, `email`, and `profile`; do not request offline access or Google business APIs. Keep client credentials in the secret manager and set `account.encryptOAuthTokens: true`. Configure the Google provider's supported `disableSignUp: true`; an unknown Google identity cannot silently create a new local user or membership. Existing users explicitly link Google through phase 06 before Google-only sign-in. Enable only the authorization-code browser flow; reject client-supplied `idToken`, scope overrides, `requestSignUp`, and arbitrary authorization parameters at the service boundary. Controlled registration and explicit account linking from phase 06 remain separate operations.

Use the supported dynamic `baseURL` with exact HTTPS host allowlisting and no unknown-host fallback. The authenticated ingress replaces forwarding/application headers. Register each approved application's full `/api/auth/callback/google` URI with Google, and bind native OAuth state to that application's origin and approved final callback in `identity.auth_flow_bindings`. Protocol state/PKCE/nonce remain native-owned; the binding stores an HMAC reference, not a duplicate OAuth state token.

Set `account.accountLinking.disableImplicitLinking: true`. Email equality alone never joins an existing user to Google; linking requires the existing user's fresh authenticated session, verified mailbox, appropriate assurance, and proof of the Google account. Do not add Google to a broad `trustedProviders` exception that accepts an unverified email. Never map Google claims to global administration or application membership.

### Passkey RP topology and user verification

Use `@better-auth/passkey`, not a parallel SimpleWebAuthn credential store. Example production hosts are `https://one.apps.example.com` and `https://two.apps.example.com`; both are controlled children of RP ID `apps.example.com`. They are placeholders to replace with actual controlled hosts before implementation. Explicitly configure `rpID`, `rpName`, and the exact allowed origins. An arbitrary unrelated domain cannot join this shared RP; new-domain support requires a separate design decision.

Host-only session cookies remain separate per application. A passkey can be used at both approved origins because they share the RP; this does not imply browser-cookie SSO or access to both applications. Registration requires a fresh usable session, active membership, and completed MFA when enabled. Configure `registration.requireSession: true`, disallow registration `createSession: true`, and never supply an unauthenticated registration user resolver.

Require discoverable credentials with `authenticatorSelection.residentKey: "required"` and user verification with `authenticatorSelection.userVerification: "required"`. Important v1.7.8 source detail: native verification uses `requireUserVerification: false`; authentication options hardcode `userVerification: "preferred"` and verification also uses false. Therefore configuration alone does not enforce this service policy. Use `registration.afterVerification` to require `verification.registrationInfo.userVerified === true` before persistence. Use `authentication.afterVerification` to require `verification.authenticationInfo.userVerified === true` before the native counter/session update. These supported hooks receive verified server results; do not trust client flags. The BFF/browser requests `userVerification: "required"` during authentication to avoid a successful authenticator ceremony that the server will reject. No imaginary `authentication.userVerification` plugin option is introduced.

Native passkey/Google session creation initializes service assurance as `primary_only` for MFA-enabled users. Passkey biometrics/device PIN prove WebAuthn user verification, but do not replace the separately enrolled TOTP/recovery factor under the confirmed requirement.

## Dependencies and prerequisites

Complete M1–M3 and use the pinned Better Auth/passkey versions from phase 01. Provision Google project consent configuration, client credentials and exact redirect URIs outside source control. Browser WebAuthn invocation belongs to first-party applications; this backend specification defines its contract without adding application frontend code. Production requires HTTPS, stable RP ID, controlled DNS, exact origins, and dependable clock synchronization.

## Database design

Google accounts/tokens, passkey credentials and native challenges/sessions stay in `auth`. Retain native ID types, credential IDs, public keys, counters, transport/device metadata, and uniqueness rules from reviewed generated SQL. Private authenticator keys never enter the server. Do not duplicate native passkey rows in TypeORM or apply generic soft deletion to them.

Reuse application memberships, flow bindings and session assurance in `identity`, plus `audit` events. Bind passkey challenge references to application, origin, ceremony, native user where known, and expiry. Native challenge cookies are host-only; do not broaden their Domain to make another app's ceremony succeed. Registration and authentication challenges are distinct purposes even when the plugin uses shared challenge storage. Account-unlink and passkey-delete operations advance the user's security epoch/revoke affected sessions according to phase 06; this is custom orchestration, not an undocumented plugin guarantee.

## API contracts

All rows are native v1.7.8 paths under `/api/auth`. Preserve the native body/status/cookie/redirect shape. Anonymous initiation means no user session; the BFF must still be authenticated and bound to an approved application. Protected management requires a usable session, active membership, and fresh assurance. Native session/token output is sensitive and stays at the BFF boundary.

| Method and native path | Request / caller | Native result and effects | Errors and audit |
| --- | --- | --- | --- |
| `POST /sign-in/social` | Anonymous initiation; service allows `provider: "google"`, approved `callbackURL`, `errorCallbackURL`, optional `newUserCallbackURL`, `disableRedirect`; reject other provider/credential/scope/signup overrides. | `{url,redirect}` for authorization-code initiation, with native authorization Location behavior retained when enabled; creates native state/PKCE flow and application binding. | Native unsupported/misconfigured-provider errors, custom wrong-origin/app denial, throttling. Audit `sign_in.started` with `google`, never authorization URL or state. |
| `GET /callback/google` | Google's query `code`, `state`, or provider `error` fields; native callback/state processing plus bound application context. Core generic callback also accepts POST, but release-one Google allowlist exposes GET only. | Native redirect and session cookies after validated provider exchange; creates/uses native account and session according to controlled linking policy; initializes assurance. | OAuth cancellation, invalid state, exchange/identity/link errors go to approved error destination with native error behavior. Audit primary success/failure and explicit linking separately. |
| `GET /passkey/generate-register-options` | Fresh usable session; optional `authenticatorAttachment` (`platform` or `cross-platform`) and `name`; disallow arbitrary registration `context`. | Native `PublicKeyCredentialCreationOptionsJSON`; stores ceremony challenge/cookie. | Native session errors, policy MFA/freshness denial. Audit `passkey.registration_started`. |
| `POST /passkey/verify-registration` | Same session/flow; `response` WebAuthn registration JSON; optional bounded `name`; reject `createSession:true`. | Native Passkey object including `id`, `name?`, `publicKey`, `userId`, `credentialID`, `counter`, device/backed-up/transport fields, `createdAt`, `aaguid?`; persists verified credential. | Invalid/missing/replayed challenge, wrong user/RP/origin, duplicate credential, failed registration, custom verified-UV failure. Audit `passkey.registered` or failure, only credential row ID. |
| `GET /passkey/generate-authenticate-options` | Anonymous or existing session; no required body/query. App-bound ceremony. | Native `PublicKeyCredentialRequestOptionsJSON`; stores native authentication challenge/cookie. BFF requests required user verification. | Native/config errors and shared throttling. Audit `passkey.authentication_started`. |
| `POST /passkey/verify-authentication` | `response` WebAuthn authentication JSON and matching host-only challenge cookie/binding. | Native `{session,user}` plus session cookie; verifies assertion, advances credential counter, and creates native session. Initializes assurance. | Native `CHALLENGE_NOT_FOUND`, `PASSKEY_NOT_FOUND`, `AUTHENTICATION_FAILED`; custom verified-UV denial before session issuance. Audit primary success/failure. |
| `GET /passkey/list-user-passkeys` | Usable session and membership. | Native Passkey array for current user; returns native metadata, never private keys. | Session/policy denial. Audit `passkey.listed` metadata only. |
| `POST /passkey/update-passkey` | Fresh usable session; `id`, nonempty trimmed `name`; ownership required. | Native `{passkey: Passkey}`; renames current user's credential. | Native ownership/not-found/validation errors, policy denial. Audit `passkey.renamed`. |
| `POST /passkey/delete-passkey` | Fresh usable session; `id`; ownership and remaining-recovery policy required. | `{status:true}`; deletes native credential. Custom orchestration revokes/invalidates affected session assurance. | Native ownership/not-found errors; reject removal of last usable recovery/sign-in method. Audit `passkey.deleted`. |

Account linking/unlinking routes are defined once in phase 06. No duplicate custom Google callback or passkey CRUD API is introduced here. Service policy failures follow phase 02; native APIError codes and statuses remain source-owned. Do not convert a failed passkey verification into another-method success automatically.

## Security considerations

Exact per-application callback checks apply even when all origins are globally trusted. Reject host/forwarding-header spoofing, flow swapping, callback laundering, ceremony swapping, and app-membership bypass. Protect generate-options GET routes against cross-site initiation using exact Origin/Fetch Metadata and trusted BFF context; challenge GETs are not cacheable. Pin the RP permanently for deployed credentials; changing it requires re-enrollment planning.

Use native signature, challenge, origin, RP-ID, credential ownership and counter checks. Synchronized passkeys can have authenticator-specific counter behavior; rely on the pinned verifier rather than imposing an invented universally increasing-counter rule. Treat anomalous verification as failure/security signal, not a reason to skip checks. Audit enrollment/deletion and send account-security notifications through SMTP. Losing all devices uses existing verified recovery methods and guarded factor recovery; support staff never manufacture a passkey or waive MFA based on email alone.

Use the same per-user credential-mutation lock/lease as native account unlinking in phase 06. Recheck remaining usable methods under the lock and retain it through native passkey deletion, policy epoch update, and any reconciliation fence. Parallel deletion and OAuth/credential unlink must not each remove the method that the other request assumed would remain.

## Implementation plan

1. Configure Google code flow, encrypted provider tokens, disabled implicit linking and provider registration for every approved host; restrict public request fields/routes.
2. Bind OAuth state/callbacks and final redirects to the authenticated application; apply membership/assurance policy to every resulting session.
3. Configure the controlled-parent RP, exact origins, session-required registration and required user verification. Add the two verified-result hooks before allowing passkey enrollment.
4. Expose native ceremony/management routes with application-bound challenges and fresh-session checks; retain native response contracts and ownership checks.
5. Add recovery-method invariants, security notifications, audit and session/epoch invalidation for credential changes.
6. Collect EXT acceptance evidence before M4 promotion, including real-browser and Google sandbox checks in the later implementation task.

## Acceptance criteria

- EXT-01: Google code sign-in completes from each approved app using its registered exact redirect URI; unknown hosts, scope overrides, client ID-token injection and cross-app callback swaps fail.
- EXT-02: Google email matching never silently links an account; explicit linking requires fresh existing-user proof and provider proof. Unverified email does not bypass linking policy.
- EXT-03: Google tokens are encrypted at rest and absent from views/logs/audit; provider claims cannot grant roles or application membership.
- EXT-04: Google primary success cannot bypass account restrictions, membership, bootstrap, or MFA; callback cancellation/failure creates no usable service session.
- EXT-05: Passkeys enroll only through a fresh usable session with verified UV; unrelated origins/RPs, unauthenticated registration, wrong-user registration, and create-session overrides fail.
- EXT-06: Authentication requires a verified UV result, valid native assertion, correct application-bound challenge, RP and origin; missing/replayed/swapped ceremonies fail.
- EXT-07: A passkey registered under the controlled shared RP can authenticate at both approved origins, but host-only cookies and application access remain independent; arbitrary domains fail.
- EXT-08: Users list/rename/delete only their own credentials; removing the last usable recovery/sign-in path is denied, including concurrent mixed-method removal, and security changes trigger defined audit/notification/revocation behavior.
- EXT-09: MFA-enabled passkey users remain `primary_only` until TOTP/recovery succeeds. Real-browser cancellation, lost-device recovery, concurrent replay and native counter behavior have recorded future evidence.

## Official references

- [Google provider](https://better-auth.com/docs/authentication/google) and [Google web-server OAuth](https://developers.google.com/identity/protocols/oauth2/web-server)
- [Account linking](https://better-auth.com/docs/concepts/users-accounts) and [Better Auth v1.7.8 social initiation](https://github.com/better-auth/better-auth/blob/v1.7.8/packages/better-auth/src/api/routes/sign-in.ts)
- [Passkey plugin](https://better-auth.com/docs/plugins/passkey), [v1.7.8 route implementation](https://github.com/better-auth/better-auth/blob/v1.7.8/packages/passkey/src/routes.ts), and [v1.7.8 options/hooks](https://github.com/better-auth/better-auth/blob/v1.7.8/packages/passkey/src/types.ts)
- [WebAuthn relying-party ID](https://www.w3.org/TR/webauthn-3/#rp-id) and [user verification](https://www.w3.org/TR/webauthn-3/#user-verification)

Static source evidence is not runtime/browser compatibility evidence. The UV hooks and shared-RP behavior are mandatory future verification gates, not claims of completed tests.
