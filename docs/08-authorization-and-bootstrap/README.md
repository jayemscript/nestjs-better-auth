# Authorization and SuperAdmin Bootstrap

Release classification: first-release requirement, M3. Last reviewed: 2026-10-15.

[Documentation index](../README.md) · Prerequisites: [Persistence](../03-database-and-persistence/README.md), [Security](../04-security-and-audit-foundations/README.md), [Identity](../06-authentication-and-account-lifecycle/README.md), [Sessions](../07-session-management/README.md). Read the operator provisioning subset of [13](../13-application-and-service-integration/README.md) during M1; full integration remains M5.

## Objective and scope

Separate shared identity, global administration, and application membership. Provide safe HTTP administration and one idempotent operator SuperAdmin bootstrap command. Use native Better Auth administration for global account actions and TypeORM only for service-specific policy. Bootstrap is never an unauthenticated HTTP endpoint.

## Architecture and design

Configure the native admin plugin with explicit access-control definitions for `user`, `admin`, and `superadmin`, with default role `user`. `superadmin` must actually exist in the plugin's custom role definition; adding its name to `adminRoles` alone is insufficient. Do not configure `adminUserIds` as an unreviewed universal bypass. Keep impersonation disabled for release one. [Native admin access control](https://better-auth.com/docs/plugins/admin).

Use single global role values, not comma-separated role combinations. App roles `member` and `app_admin` belong only to active application memberships. Define a static permission map in service code and configure equivalent native permissions for server-side calls. There is no dynamic role editor or duplicate credential/user table.

Reuse the selected Nest library's session/global-role decorators where they match this policy, composing the authoritative service guard from 02. `@Roles` checks native global roles, not application membership, mobile transport, current security epoch or MFA assurance. Do not enable organization decorators as a substitute for application roles. The same permissions apply to web cookies and native mobile bearer sessions; a public mobile client identifier never grants a role.

| Actor | Permitted behavior |
| --- | --- |
| `user` + member | Own account/sessions within policy; no administration |
| `user` + app_admin | Read own app and memberships; add/update/remove only `member` memberships in that app; no global user search or account actions |
| `admin` | Read global human users; ban/unban/delete ordinary `user` accounts; manage applications and memberships, including app_admin assignments; cannot change global roles or act on admin/superadmin accounts |
| `superadmin` | Global role assignment and guarded actions on all human users; same application administration; last-SuperAdmin invariant applies |

### Registered identity permission names

The following is the exact release-one permission catalog consumed by `/internal/v1/authorization/check` in 13. It expresses identity-service capability, not downstream business-resource ownership. Reject unknown names rather than dynamically interpreting strings.

| Permission names | Granted to and scope |
| --- | --- |
| `account:read`, `account:manage`, `sessions:read`, `sessions:revoke` | Every active member for its own identity; operation-specific freshness and MFA checks still apply |
| `application:read`, `memberships:read`, `memberships:manage` | Own app_admin in the caller-bound app; global admin/superadmin can target approved apps; app_admin member-only mutation limits still apply |
| `application:manage` | Global admin/superadmin, with provisioning/last-SuperAdmin guards |
| `users:read`, `users:moderate` | Global admin/superadmin; admin moderation targets only global `user` accounts |
| `users:roles` | SuperAdmin, with last-SuperAdmin guard |
| `sso:manage`, `service-clients:manage` | SuperAdmin only; corresponding provider/key workflow guards remain mandatory |

App `member` receives only the own-account/session set; `app_admin` adds the app-scoped set. Global roles add the stated capabilities to an existing active app membership; they do not remove target, freshness, or assurance checks. A bare permission-name match is never sufficient to authorize a specific target user or application. Translate this catalog to native admin resource/actions explicitly rather than passing service permission strings as native plugin statements.

Global roles do not silently create application membership. An administrator needs active membership in the originating administration application. All administration, including app_admin reads, requires enrolled MFA, complete assurance, and the relevant permission. Every administrative mutation additionally requires a five-minute native session and a TOTP/recovery success within five minutes; even an ordinary global admin must enroll MFA before administration. Feature-specific contracts may also require that freshness for especially sensitive reads.

Server-only typed native operations include listing/getting users, setting role, banning/unbanning, removing users, and session revocation. External `/api/auth/admin/*` remains denied. Authenticate and authorize in Nest first, record durable operation intent, then invoke native APIs inside the trusted operation capability described in 06. Do not grant admin powers to an untrusted header, service key without user authority, or app_admin membership.

### Serialization and partial failure

Persist `identity.admin_operations` before privileged writes. It records actor, target, app, idempotency key, redacted intent, stage, lease, and outcome. Use a PostgreSQL advisory lock plus operation lease around serialized global privilege mutations, bans, deletion, and bootstrap. The coordination connection holds the lock while native work uses its independent pool. On crash its lock is released; a successor must inspect/reconcile the expired operation before allowing a conflicting operation.

All global role/lifecycle mutations must use this service path, including operator recovery tooling. Deny raw native HTTP and require the internal operation capability on typed admin calls. The PostgreSQL `auth` runtime role still enforces its granted native-table boundary; prevent arbitrary database writers operationally. Do not claim the identity transaction and native action are atomic.

Protect the last eligible SuperAdmin under the shared lock. Count active, verified human SuperAdmins with active bootstrap state, enrolled MFA, non-banned lifecycle, and an active administration-app membership. Do not ban/delete/demote such an account or suspend/remove its final administration membership unless another eligible account remains. Also forbid an application status change that would remove the last eligible administration access. Pending bootstrap accounts do not count as replacements. A partially completed promotion grants no usable privilege until policy is active; reconcile before releasing conflicting changes.

### Bootstrap lifecycle

One future command, `identity:bootstrap-superadmin`, accepts `--email`, `--name`, and `--application-id` for an already operator-provisioned administration application. The command uses the validated runtime/migration-free composition and never starts public HTTP. SMTP and a legitimate verification flow must be configured. Prefer a trusted operator session/secret manager to ad hoc environment dumping.

1. Acquire the bootstrap/global-privilege lock and persist a `bootstrap_runs` record keyed by the bootstrap identity and application. Reject an existing unrelated user with the same email; never promote or reset it automatically.
2. Generate a high-entropy one-time password in memory. Output it exactly once to an interactive TTY, or to an explicitly configured approved secret-manager sink. Refuse non-interactive stdout/CI output without that sink. Never write a repository file, shell history argument, audit payload, log, or ordinary environment value containing it.
3. Call typed native `createUser` with `role:"superadmin"` in its supported requestless/headerless operator form, under the narrow in-process capability. Unlike `setRole`, this first-account path does not require an existing administrator session. Resolve canonical application context through trusted configuration, not forged actor headers. Native role alone grants no effective access: persist `account_policies.bootstrap_state=password_change_required`, required membership, and the operation's intended role before allowing sign-in. Missing policy denies access. Native `emailVerified` remains false.
4. Request native email verification and queue SMTP to the supplied address. Email possession is established through the normal verification link. The generated password plus verification permits only the restricted bootstrap sign-in, `/me`, password change, the subsequent MFA-enrollment operations, and logout.
5. After successful native password change, move to `mfa_enrollment_required`, revoke old sessions, and use the resulting/new native login session. Do not infer password replacement from a submitted body or a failed response.
6. After native TOTP enrollment and successful TOTP proof, verify the staged native `superadmin` role, ensure the administration membership is active, set bootstrap state `active`, advance epoch, and require a fresh sign-in plus enrolled factor before normal administration. Do not call a session-required native role mutation as an unauthenticated first administrator.
7. Mark completion and audit the non-secret outcome. Rerunning an existing completed run returns already-complete without credential output/reset. Rerunning a pending run resumes only confirmed missing non-secret stages; it never silently resets a delivered password. A password lost before safe delivery uses the native verified-email recovery process, retaining bootstrap restrictions.

If the native user creation succeeds but policy persistence fails, the missing policy blocks sign-in/protected access. Reconcile that exact run/user linkage rather than searching by email and promoting a match. A sink failure after creation is a recorded incomplete run, not permission to print a secret into logs. A second concurrently started command must observe the first run and cannot create a second account or compete for the role. Operator disaster recovery is a reviewed procedure in 16, not an authentication bypass.

## Dependencies and prerequisites

Use native admin access control plus the existing TypeORM service persistence and database locking primitives. SMTP, authoritative policy, session assurance, and minimal app/service provisioning must be ready. TOTP enrollment in 09 is required to complete bootstrap; phases 08 and 09 are promoted together as M3.

The native APIs' auth and transaction semantics are source-dependent. The trusted operation context authorizes narrow orchestration, never arbitrary code that can choose any native method. Before implementation, bind each operation to an explicit typed API and audit permission; retain native actor headers when the method requires them.

## Database design

The admin plugin owns native user role/ban fields and any plugin session fields in `auth`. Do not duplicate role or ban truth in a second user table. TypeORM `identity.account_policies` owns lifecycle, epoch, bootstrap state, kind (`human` or `service`), and optimistic version; it may hold a short-lived pending operation restriction while native state changes.

`identity.applications` contains UUID ID, unique slug, name, exact public origin, status, self-registration policy, and version. `identity.application_memberships` contains application/user references, `member|app_admin`, `active|suspended`, and version with unique `(application_id,user_id)`. Reuse UUIDv7/version/timestamps from the existing base where appropriate; native user IDs retain their format. `admin_operations` and `bootstrap_runs` provide operation/lease/idempotency state as specified in 03. Audit remains in `audit`.

Application origin/slug changes require coordinated ingress/provider registration and are not supported by the ordinary patch endpoint in release one. Create applications disabled, provision their service binding and exact origin through the operator path in 13, then activate. Deleting an application means disabling it in release one, retaining referenced audit/operation evidence; destructive application purge is deferred.

## API contracts

All routes in this section are proposed Nest APIs using 02's envelope, pages, ID types, and errors. `AdminUserView` is `UserView + {globalRole,banned,banExpiresAt?,lifecycle,bootstrapState,version}` and never includes session/provider tokens or password hashes. User list/get returns only human accounts; service-account management stays in 13. Membership operations also reject `kind=service` targets.

Use ETag/If-Match on every versioned administrative mutation. Missing If-Match is 428; stale is 412. Use `Idempotency-Key` on operation-backed mutations, including creates, role/ban/deletion, and membership/application changes; changed actor/body under a reused key is 409. Authorization runs before result replay. Custom user versions advance through account policy under the operation lock; native changes outside orchestration are unsupported. A bodyless 204 has no envelope.

Read errors: 401 invalid session, 403 permission/MFA/bootstrap denial, 404 missing/inaccessible target, 503 required storage unavailable. Mutations add 400 validation, 409 invariant/idempotency conflict, 412/428 concurrency errors, and 429 throttling. Forbidden global targets are 403; inaccessible cross-app resources are 404. Audit every privileged read/mutation outcome with redacted actor/target/app/operation IDs. Do not return native raw errors/tokens through these custom APIs.

| Method/path | Caller and input | `data`/status | Side effects and event |
| --- | --- | --- | --- |
| GET `/api/v1/admin/users` | Global admin/superadmin; `limit?,cursor?,email?` exact email filter only | 200 page of AdminUserView | Authorized global human-user read; `admin.users_read` |
| GET `/api/v1/admin/users/:userId` | Global admin/superadmin; opaque native user ID | 200 AdminUserView + ETag | No credential export; `admin.user_read` |
| PUT `/api/v1/admin/users/:userId/role` | SuperAdmin; `{role:"user"|"admin"|"superadmin"}`, If-Match, Idempotency-Key | 202 `{operationId,state}` | Serialized native setRole, epoch/revocation, version advance; promotion to admin/superadmin requires verified email, active policy, and enrolled MFA; `admin.role_changed` |
| PUT `/api/v1/admin/users/:userId/ban` | Admin for ordinary user, or superadmin; `{reason,expiresAt?}`; same concurrency headers | 202 operation | Native ban + immediate policy/epoch restriction and revocation; `admin.user_banned` |
| DELETE `/api/v1/admin/users/:userId/ban` | Same target authority/headers, no body | 202 operation | Native unban; remove ban-related restriction only, never undo deletion/bootstrap/membership policy; `admin.user_unbanned` |
| DELETE `/api/v1/admin/users/:userId` | Same target authority/headers; `{reason}` | 202 operation | Durable native/custom deletion saga from 06; `admin.user_deletion` |
| GET `/api/v1/admin/operations/:operationId` | Originating authorized actor or superadmin; same application context | 200 `{id,state,stage,errorCode?,createdAt,updatedAt}` | No payload/secrets/stack; inaccessible operation 404; `admin.operation_read` |
| GET `/api/v1/admin/applications` | Global admin/superadmin; `limit?,cursor?` | 200 ApplicationView page | App registry read; `admin.applications_read` |
| POST `/api/v1/admin/applications` | Global admin/superadmin; `{slug,name,publicOrigin,selfRegistration:false}`, Idempotency-Key | 201 ApplicationView + ETag; starts disabled | Exact origin/slug uniqueness (409); creates registry only, provisioning/activation required; `application.created` |
| GET `/api/v1/admin/applications/:applicationId` | Global admin/superadmin or own app_admin | 200 ApplicationView + ETag | Inaccessible app 404; `application.read` |
| PATCH `/api/v1/admin/applications/:applicationId` | Global admin/superadmin; `{name?,status?:"active"|"disabled",selfRegistration?}`, If-Match, Idempotency-Key | 200 ApplicationView + ETag | Activation requires provisioned binding; disabling checks last-SuperAdmin access; `application.updated` |
| GET `/api/v1/admin/applications/:applicationId/memberships` | Global admin/superadmin or own app_admin; `limit?,cursor?` | 200 MembershipView page | Read only visible app assignments; `membership.listed` |
| GET `/api/v1/admin/applications/:applicationId/memberships/:userId` | Same reader authority | 200 MembershipView + ETag | No cross-app disclosure; `membership.read` |
| POST `/api/v1/admin/applications/:applicationId/memberships` | Global admin/superadmin or own app_admin; `{userId,role,status:"active"}`, Idempotency-Key | 201 MembershipView + ETag | Existing human target only; duplicate pair 409. app_admin can assign only member; `membership.created` |
| PATCH `/api/v1/admin/applications/:applicationId/memberships/:userId` | Same authority; `{role?,status?:"active"|"suspended"}`, If-Match, Idempotency-Key | 200 MembershipView + ETag | app_admin can modify only member→member; blocks inherited privilege/self-promotion and last-SuperAdmin lockout; policy invalidates immediately; `membership.updated` |
| DELETE `/api/v1/admin/applications/:applicationId/memberships/:userId` | Same authority/headers, no body | 204 | app_admin cannot delete app_admin assignments; last-SuperAdmin check; removes assignment and related app assurance; `membership.deleted` |

The global-role operation never treats an app_admin role as a global role. Membership status changes affect access without altering native identity ownership. Global admin operations on admins/superadmins require superadmin; membership administration cannot remove the final eligible SuperAdmin's access indirectly. Native `removeUser` forbids an administrator deleting itself: the custom admin deletion route returns 400 `USE_SELF_DELETION` for that case and directs the authenticated user to guarded `/api/v1/me` deletion. Direct password assignment/reset or impersonation through the native admin HTTP API is not exposed. Normal password recovery is the approved credential-recovery channel.

202 operation states are `pending|running|succeeded|failed`; a failed operation includes only a safe stable code. Map the storage phase in 03 explicitly: pending to pending, native_applied/policy_applied to running, completed to succeeded, and failed to failed. Resource lifecycle such as deleting is a separate field, never another operation-state enum value. A synchronous custom resource mutation returns success only after its TypeORM commit and durable audit evidence. If native/global orchestration has committed only partially, return an operation state rather than pretend it rolled back. The implementation may finish an operation before 202 is returned; clients still use the operation contract.

## Security considerations

Check permissions against authoritative native role and current membership/epoch on each request; a session-cached role is insufficient. Roles, memberships, lifecycle, and bootstrap transitions are not caller-writeable profile fields. Service keys authenticate applications and never make their holder a global administrator.

Privileged mutations require durable audit intent before side effects; if audit preparation is unavailable, fail safely. Redact free-text reasons to bounded validated text and exclude secrets. Administrative user search is exact-email-only in release one to limit broad enumeration; collection pagination is still bounded. Native provider scopes and arbitrary metadata are not accepted by these routes.

Global banning, deletion, demotion, and membership suspension have different semantics. Unban cannot reactivate a deleting account or recreate a membership. MFA recovery cannot clear bootstrap requirements. Do not count a native `superadmin` role alone as proof that an operator bootstrap completed. Bootstrap's temporary secret must never flow through SMTP templates; SMTP proves email possession, while the password is delivered through the selected secure operator channel.

## Implementation plan

1. Implement the static global/app permission map, native admin roles, no impersonation, and guards using authoritative sessions/policy.
2. Add schema-qualified application/membership repositories, optimistic versions, pagination, and the application provisioning/activation preconditions from 13.
3. Establish durable operation intent, idempotency, advisory-lock/lease coordination, last-SuperAdmin checks, and crash reconciliation before any global mutations.
4. Add guarded HTTP administration, narrowly permitted typed native calls, response projection, audit, and external native-admin denial.
5. Implement the operator bootstrap command with exact-run ownership, safe one-time output, legitimate email verification, and restricted policy states.
6. Integrate native password-change and TOTP completion hooks from 09; update the resulting session/epoch and require fresh privileged sign-in before activation is usable.
7. Document operator recovery without bypass paths. Collect AZ evidence together with the M3 MFA gates before permitting production administration.

## Acceptance criteria

- AZ-01: A table-driven permission review proves user/member/app_admin/admin/superadmin boundaries; service keys and client role fields grant no user administration.
- AZ-02: Every protected request checks current role, active membership, lifecycle, epoch, and assurance; native `/admin/*`/impersonation bypass attempts fail.
- AZ-03: Membership CRUD enforces own-app/member-only app_admin limits, uniqueness, ETags, idempotency, and immediate suspension effects without leaking other apps.
- AZ-04: Global native role/ban/deletion operations record durable intent and audit; injected cross-pool failures remain restricted and reconcile to an explicit operation result.
- AZ-05: Concurrent demotion/ban/delete/membership/app-disabling operations cannot remove the last eligible SuperAdmin's administration access; pending accounts never count as replacements.
- AZ-06: Repeated and concurrent bootstrap creates one owned native identity/run, refuses unrelated existing email, never resets a completed account, and never discloses the password through non-interactive stdout/logs/files.
- AZ-07: Bootstrap requires real email possession, successful password replacement, TOTP enrollment/proof, and a fresh factor-backed sign-in before normal access; every intermediate native/Nest mutation obeys the restricted allowlist.
- AZ-08: Bootstrap sink/database/SMTP failures leave a safe resumable state; reconciliation uses the exact run/user association, and native recovery restores access only through verification while retaining pending bootstrap/MFA obligations.

These criteria specify later implementation verification; no commands, tests, builds, or installations were executed for them during documentation authoring.

## Official references

- [Better Auth admin plugin and custom access control](https://better-auth.com/docs/plugins/admin)
- [Versioned admin routes](https://github.com/better-auth/better-auth/blob/v1.7.8/packages/better-auth/src/plugins/admin/routes.ts)
- [Native user lifecycle](https://better-auth.com/docs/concepts/users-accounts)
- [Two-factor plugin](https://better-auth.com/docs/plugins/2fa)
- [PostgreSQL advisory locks](https://www.postgresql.org/docs/current/explicit-locking.html#ADVISORY-LOCKS)
