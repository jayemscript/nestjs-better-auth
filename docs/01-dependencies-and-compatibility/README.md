# Dependencies and Compatibility

Release classification: first-release prerequisite, M0. Last reviewed: 2026-10-15.

[Documentation index](../README.md) · Next: [Common infrastructure](../02-common-infrastructure/README.md)

## Objective and scope

Define the smallest compatible dependency baseline for the planned service. This is a package change specification; no packages are installed, removed, upgraded, or executed by this documentation task.

## Architecture and design

### Observed baseline

The manifest specifies ranges; the lockfile and installed metadata resolve the following relevant versions. An installed package is not proof that the application starts.

| Package | Manifest | Observed resolved version | Treatment |
| --- | --- | --- | --- |
| Nest common/core/platform-express | `^11.0.1` | 11.2.7 | Retain Nest 11 and Express adapter |
| Express via platform-express | Transitive | 5.2.1 | Use Express 5 named wildcard routing |
| `@nestjs/config` | `^4.0.4` | 4.0.4 | Reuse configuration boundary |
| `@nestjs/typeorm` | `^11.0.1` | 11.0.3 | Reuse database integration |
| `typeorm` | `^1.0.0` | 1.1.2 | Retain initially; verify migration/API compatibility later |
| `pg` | `^8.21.0` | 8.23.1 | Reuse for supported Better Auth connection |
| `redis` | `^6.0.0` | 6.3.0 | Installed but not required for release-one auth storage |
| TypeScript | `^5.7.3` | 5.9.3 | Retain NodeNext and validate tooling conversion |
| `uuid` | `^14.0.0` | 14.0.3 | Reuse UUIDv7 for custom entities |
| Better Auth | Absent | Absent | Future direct dependency |
| Nodemailer | Absent | Absent | Future SMTP-only transport dependency |

Existing config, Swagger, throttler, validation, scheduling, and test packages should be reused. Installed `@nestjs/jwt`, bcrypt, argon2, and SimpleWebAuthn do not justify building parallel identity functions. Do not remove them automatically during this documentation task. During implementation, remove a direct dependency only after confirming it has no service-owned use; plugin-transitive dependencies remain owned by the plugin.

### Proposed additions

| Package | Baseline researched | Purpose and boundary |
| --- | --- | --- |
| `better-auth` | 1.7.8 | Core plus bundled username, admin, two-factor, magic-link, email-OTP, bearer, and one-time-token plugins |
| `@thallesp/nestjs-better-auth` | 2.8.0 | Selected Nest integration: `AuthModule.forRootAsync`, `AuthService`, native handler middleware, and Nest authentication guard; no second manual handler |
| `@better-auth/passkey` | 1.7.8 | Native passkey implementation |
| `@better-auth/sso` | 1.7.8 | Inbound OIDC and SAML federation |
| `@better-auth/api-key` | Same compatible Better Auth release line; pin after metadata review | Service credential lifecycle; sessions from keys disabled |
| `nodemailer` | 10.1.1 | SMTP transport only |
| `@types/nodemailer` | Matching compatible type declarations if required | Development typing only |
| `@nestjs/terminus` | Nest 11-compatible release, pin before installation | Liveness/readiness composition |
| `helmet` | Express 5/Node 22-compatible release, pin before installation | Security headers at the HTTP boundary |
| `auth` CLI | Exact version compatible with the chosen Better Auth release | Development-only schema SQL inspection/generation in later implementation |

Versions above are research observations, not a promise of future availability or permission to use floating `latest`. Before future installation, record exact versions, peer requirements, package integrity, relevant advisories, and license/deployment constraints. Pin the reviewed result in the manifest/lockfile. Do not replace the chosen persistence path with a community adapter as an installation workaround.

### Compatibility evidence

- Published and installed `@nestjs/typeorm` 11.0.3 metadata permits Nest 10/11 and TypeORM `^0.3.0 || ^1.0.0-dev`. The existing stable TypeORM 1.1.2 resolution must be checked with the package manager's semver interpretation and actual APIs during implementation; do not declare it broken merely because it is 1.x.
- TypeORM 1.1.2 declares Node `^20.19.0 || ^22.13.0 || >=24.11.0`. The selected Nest bridge 2.8.0 requires Node `>=22.22.1`, so choose a pinned supported Node 22 patch at least 22.22.1; this stricter bridge floor supersedes TypeORM's lower Node 22 minimum. Schedule LTS upgrades separately.
- Published bridge 2.8.0 peers include Nest `^11.1.6`, TypeScript `^5.9.2`, Express `^5.1`, and Better Auth `>=1.5 <2`. Observed Nest 11.2.7, TypeScript 5.9.3, Express 5.2.1, and proposed Better Auth 1.7.8 satisfy those ranges; actual build/runtime compatibility remains unverified. Use the selected bridge instead of a parallel hand-mounted handler or community persistence adapter.
- Better Auth 1.7.8 and the researched standalone plugin packages publish ESM. The Express integration explicitly requires ESM. `module: nodenext` alone does not turn `.ts` output into ESM when the package has no `type: module`.
- Better Auth's PostgreSQL path accepts `pg.Pool`. It does not require a TypeORM adapter. Pools have separate connection budgets and transactions.
- Web and native iOS/Android authentication are first-release requirements. The bundled `bearer` and `oneTimeToken` plugins provide native session transport; no Expo-only integration, OAuth token issuer, mobile client secret, JWT session, or refresh-token pair is required. Primary authentication and enrolled MFA complete in a hosted system browser before the [mobile handoff](../14-mobile-authentication/README.md).
- For the pinned release, configure `bearer({ requireSignature: true })` and `oneTimeToken({ disableClientRequest: true, storeToken: 'hashed', expiresIn: 1, disableSetSessionCookie: false, setOttHeaderOnNewSession: false })`. One-time-token expiry is in minutes. Native generate/verify operations are server API calls behind the gateway, never directly exposed HTTP endpoints. Validate the signed bearer returned by native verification through typed `auth.api.getSession` using the bearer plugin; do not read `auth.session` by token or treat a signature as sufficient authorization.
- No runtime build, type check, tests, database connection, or external-provider validation was performed. Future verification is a release prerequisite.

## Dependencies and prerequisites

The [overview](../00-project-overview/README.md) supplies requirements. Before feature implementation, identify the runtime image digest, package-manager version, PostgreSQL target version, SMTP transport settings, and externally supplied application origins. Use PostgreSQL 18 as the proposed new-deployment baseline unless the actual platform supplies another supported version; validate selected SQL against the real server rather than assuming its version.

Keep package-manager stores and caches outside the repository. Never create `.pnpm-store` inside any repository. No dependency installation commands are executed here.

## Database design

No new table is needed for dependency management. The chosen package/plugin configuration determines native schema generation in phase 03. Record the Better Auth/plugin version and configuration fingerprint in each future migration's review notes. The database is not allowed to infer its schema from whatever package happens to be deployed.

## API contracts

This phase adds no endpoint. It constrains future contracts: bridge-mounted Express 5 native routes under `/api/auth/*`, ESM runtime exports, and pinned plugin API shapes. Construct the native instance through `AuthModule.forRootAsync`, resolving the asynchronous native adapter/provider first, and consume it through `AuthService`. Do not register another manual `toNodeHandler`. Do not expose a nonexistent HTTP endpoint for a server-only `auth.api` method. Keep versioned native source references with contracts that rely on source-level behavior. Runtime SSO field encryption decorates the supported native PostgreSQL adapter; schema inspection uses the identical enabled plugins/schema/options with an undecorated native pool, as specified in 03, because the CLI cannot infer PostgreSQL SQL generation from an arbitrary runtime adapter factory.

## Security considerations

Do not enable every dependency lifecycle script. Inventory native build requirements, then explicitly permit reviewed scripts in later package-manager configuration. Do not use `--dangerously-allow-all-builds` as the deployment default. Do not put SMTP/database/IdP secrets into install commands, build arguments, images, or example package metadata.

Retain Better Auth's password hashing contract initially; bcrypt and argon2 are not alternative password stores. Never let a library upgrade silently enable a new native auth route outside the route allowlist.

## Implementation plan

1. Capture the existing manifest and lockfile baseline; identify unused packages without changing functionality yet.
2. Check exact release metadata and advisories for all additions; select one matching Better Auth/plugin line and lock the CLI/tooling versions.
3. Convert the application package to ESM in the future implementation: package type, relative `.js` specifiers, runtime-safe imports, and removal/replacement of unresolved `src/*` runtime aliases.
4. Convert migration entrypoints and Makefile commands to an ESM-aware loader/compiled-JS workflow. Keep Jest/ts-jest configuration coherent with that choice; do not assume existing CommonJS transforms still work.
5. Pin Node/pnpm/container versions and configure any package store outside the checkout. Review lifecycle scripts individually.
6. Install the reviewed changes only in a later authorized coding task. Record actual compatibility results; repair the minimum incompatible surface before proceeding to M1.

## Acceptance criteria

- DEP-01: Exact installed and proposed versions, Node engines, plugin peers, and module format are recorded separately.
- DEP-02: ESM imports, migration tooling, development startup, production startup, and future test tooling have one documented runtime strategy.
- DEP-03: Supported PostgreSQL integration is used without adding a TypeORM authentication adapter.
- DEP-04: No duplicate password, passkey, JWT-session, or email framework is introduced.
- DEP-05: The eventual package install and container build use pinned tooling and reviewed lifecycle scripts, with no repository-local package store.
- DEP-06: Runtime verification is recorded as pending until performed in a later implementation task.
- DEP-07: The pinned bundled bearer/one-time-token interfaces, signed bearer round-trip, server-only handoff wrappers, and platform-neutral iOS/Android integration are covered by future verification; native credentials share web's underlying session and policy model.
- DEP-08: Bridge 2.8.0 peers and Node >=22.22.1 are met, only one native handler is registered, and future checks cover async provider initialization, middleware order, explicit composite guards, and retained native response contracts.

## Official references

- [Better Auth installation](https://better-auth.com/docs/installation)
- [Selected Nest integration repository](https://github.com/thallesp/nestjs-better-auth)
- [Nest bridge 2.8.0 package metadata](https://registry.npmjs.org/@thallesp%2fnestjs-better-auth/2.8.0)
- [Better Auth Express integration](https://better-auth.com/docs/integrations/express)
- [Better Auth PostgreSQL integration](https://better-auth.com/docs/adapters/postgresql)
- [Better Auth bearer plugin](https://better-auth.com/docs/plugins/bearer)
- [Better Auth one-time-token plugin](https://better-auth.com/docs/plugins/one-time-token)
- [Nest TypeORM integration](https://docs.nestjs.com/data/typeorm)
- [Nest TypeORM 11.0.3 package metadata](https://registry.npmjs.org/@nestjs%2ftypeorm/11.0.3)
- [TypeORM 1.1.2 package metadata](https://registry.npmjs.org/typeorm/1.1.2)
- [Better Auth 1.7.8 package metadata](https://registry.npmjs.org/better-auth/1.7.8)
- [Nodemailer SMTP transport](https://nodemailer.com/smtp)
- [Node.js ESM packages](https://nodejs.org/api/packages.html#type)
