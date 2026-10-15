# Future Enhancements and Extension Boundaries

Release classification: optional future work and post-launch production enhancements. Last reviewed: 2026-10-15.

[Documentation index](../README.md)

## Objective and scope

Preserve extension paths without adding unnecessary infrastructure or expanding the first-release identity boundary. Every feature listed in the current repository README remains required for release one; none is deferred by this document.

Shared identities support multiple approved web and native iOS/Android applications in the first release. The mandatory [mobile authentication contract](../14-mobile-authentication/README.md) covers browser-based sign-in, verified app-link handoff, secure device session storage, and MFA parity. Organization tenancy, independent tenant credential realms, unrelated-domain federation as an identity provider, and native desktop authentication require separate designs before implementation. This document defines their entry conditions, ownership direction, and compatibility constraints rather than inventing unapproved endpoint/schema details.

## Architecture and design

### Production enhancements after the first release

| Capability | Trigger and approach | Compatibility constraint |
| --- | --- | --- |
| Separate delivery workers | Measured queue latency or API resource contention; keep the same PostgreSQL job/lease/expiry contract | Native callbacks and mail semantics unchanged; no new challenge issuer |
| Distributed abuse storage | Measured PostgreSQL counter contention; evaluate atomic Redis custom storage | Implement native atomic consume expectations; do not silently move sessions/challenges into secondary storage |
| Telemetry export/tracing | Operational need for cross-service diagnosis; vendor-neutral instrumentation with secret redaction | No payload/cookie/assertion capture; bounded identifiers and sampling |
| Risk-based challenges | Demonstrated fraud/abuse need; enrich existing policy with reviewed risk signals | Never silently waive enrolled MFA or treat device fingerprints as identity |
| Enhanced audit integrity | Stronger compliance/forensic requirement; external append-only export and retention evidence | Database append permissions alone are not cryptographic tamper proofing |
| Cache/read scaling | Measured validation throughput bottleneck | Document bounded revocation delay before accepting any positive cache; strong initial revocation semantics otherwise remain |
| Advanced mail operations | Relay/operator feedback needs; standard SMTP bounce mailbox processing where appropriate | No provider SDK/email framework without a new user-approved requirement |

### Optional product capabilities

| Capability | Intended native foundation | Custom boundary and decisions required before adoption |
| --- | --- | --- |
| Organization tenancy | Better Auth organization plugin | Membership lifecycle, organization-to-app access, tenant-scoped roles, invitations, and non-bypassable tenant context |
| Fully isolated tenant identities | Separate realm/configuration/deployment strategy | Whether same email is distinct, separate secrets/databases, tenant discovery, operations and migration ownership; not equivalent to an organization ID |
| Native desktop clients | Existing mobile gateway/system-browser and native bearer patterns as a starting point | Desktop-specific redirect ownership, secure OS credential storage, supported platforms, installation trust, and MFA/revocation acceptance; mobile is already required in release one |
| OAuth/OIDC identity provider | Better Auth OAuth provider plugin | Client registration, consent UI, grants, scopes/audiences, keys/JWKS, code/refresh rotation, revocation, and client authentication |
| Service OAuth client credentials | Supported provider grant capabilities after version verification | Machine principal model, audiences, grant restrictions, key rotation, and migration from existing API keys |
| SCIM/directory provisioning | Better Auth SCIM/SSO capabilities after version review | Authoritative workforce identifiers, deprovisioning SLA, organization trust, paired directory mutation boundaries |
| Assisted MFA recovery | Native factor reset operations under controlled orchestration | Proof policy, delayed notifications, operator approvals, irreversible recovery evidence; email alone remains insufficient |
| Enterprise global logout | Supported SSO SLO and provider-specific behavior | Signed logout verification, native/custom session mappings, app propagation, failure semantics; local signout still makes no global logout promise |
| Regional deployment | PostgreSQL/infra design informed by residency and latency requirements | Single-writer consistency, replay state, session revocation, key distribution, recovery, and region authority |

No new plugin is activated merely because it is listed here. Re-evaluate its current official documentation, exact version, maintenance, and security model when the trigger arises.

## Dependencies and prerequisites

Complete the first-release acceptance matrix and gather production measurements before performance-driven expansion. Product-driven capabilities require a separately approved scope and concrete deployment/user requirements. Preserve the existing module boundaries and native ownership rather than replacing them with a parallel identity system.

Organization adoption depends on app membership and role contracts; a provider rollout depends on client/consent/audience requirements; regional operation depends on recovery and consistency objectives. Do not let a future feature create a circular prerequisite for the first release.

## Database design

No future table or migration is created in release one solely for speculative capabilities. Keep native plugin tables in `auth`, service-owned policy/application data in `identity`, and security evidence in `audit`.

When organizations are approved, preserve shared native user IDs and add explicit organization/app relationships through the plugin and service-owned policy; do not overload current `applicationId` to mean tenant. Fully isolated tenant identities require a separate realm design and potential data migration, not a nullable organization column.

Redis adoption must explicitly state which data it stores. Native `secondaryStorage` can change where sessions and verification records live, so enabling it for performance is an architecture change. Keep PostgreSQL verification/strong revocation semantics until the reviewed replacement proves equivalent atomic operations and failure behavior.

## API contracts

This phase adds no endpoint, public type, token format, claim, or route promise. Reserved future concepts are not live APIs. Keep current `/api/auth`, `/api/v1`, and `/internal/v1` contracts stable; introduce versioned additions when a concrete capability requires incompatible behavior.

Before a future API is accepted, specify caller identity, tenant/app authority, request/response/errors, compatibility, migration, audit, rate limits, retention, and deprecation. Do not claim that the current API key is an OAuth client-credentials token, a native session is a JWT, or inbound OIDC makes this service an OIDC issuer.

## Security considerations

Every extension must preserve current authentication ownership, enrolled-MFA enforcement, app binding, least-privilege schemas, SMTP-only delivery, and documented revocation behavior unless a new requirement explicitly revises them. New federation/tenancy boundaries need threat modeling before exposing routes.

Dynamic roles, arbitrary tenant origins, and provider self-registration must not bypass current SuperAdmin trust administration. Additional third-party SDKs or hosted services need a concrete value justification, secret/permission model, and integration acceptance.

## Implementation plan

1. Record the measurable operational trigger or approved product need.
2. Review the relevant current official plugin/version capabilities; identify native versus custom responsibilities.
3. Write a focused architecture decision and compatibility/migration plan extending this documentation.
4. Specify the new API/data/policy boundary and preserve current callers or publish a controlled version change.
5. Add security, concurrency, recovery, and rollout acceptance scenarios before enabling the feature.
6. Implement only the approved capability; deploy behind an explicit flag with operational evidence and rollback rules.

## Acceptance criteria

- FUT-01: No README first-release requirement is incorrectly deferred here; web and native iOS/Android authentication are release-one requirements, while native desktop is future work.
- FUT-02: No speculative plugin, production entity, endpoint, or infrastructure dependency is introduced by this phase.
- FUT-03: Organization tenancy, identity realms, app membership, and OAuth/OIDC issuer roles remain distinct concepts.
- FUT-04: Any future change to revocation, session/challenge storage, MFA, or trust boundaries requires explicit documentation and migration/acceptance evidence.
- FUT-05: Future work is triggered by a concrete requirement or measured constraint, with native capabilities verified before custom implementation.

## Official references

- [Better Auth organizations](https://better-auth.com/docs/plugins/organization)
- [Better Auth OAuth provider](https://better-auth.com/docs/plugins/oauth-provider)
- [Better Auth API keys](https://better-auth.com/docs/plugins/api-key)
- [Better Auth bearer sessions](https://better-auth.com/docs/plugins/bearer)
- [Better Auth enterprise SSO](https://better-auth.com/docs/plugins/sso)
- [Better Auth database and secondary storage](https://better-auth.com/docs/concepts/database)
- [Nest queues](https://docs.nestjs.com/techniques/queues)
