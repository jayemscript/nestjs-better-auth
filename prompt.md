# Role

Act as an **Expert Senior Software Architect** specializing in NestJS, Better Auth, PostgreSQL, TypeORM, authentication, identity management, and secure backend architecture.

# Project Overview

Design a reusable **Authentication and Identity Backend Service** built with:

- NestJS
- Better Auth
- PostgreSQL
- TypeORM
- REST APIs / HTTP

The service must be modular, secure, scalable, maintainable, and reusable across different application architectures, including microservices and standalone applications.

# Rules

- **Do not write or implement application code.**
- **Do not run builds, lint checks, or tests.**
- **Do not create `.pnpm-store`.**
- You may inspect existing source files, configurations, dependencies, and project structure.
- Your responsibility is strictly **architecture planning and Markdown documentation**.

# Instructions

1. Read and analyze the existing `README.md` to understand the project's current scope, features, and objectives.
2. Inspect the existing project structure, especially:
   - Database module: `C:\personal-projects\nestjs-better-auth\src\database`
   - Common utilities: `C:\personal-projects\nestjs-better-auth\src\common`
3. Reuse existing architectural foundations where appropriate. Avoid proposing unnecessary duplication or conflicting implementations.
4. Identify missing features, security requirements, architectural improvements, and opportunities to extend the current README specifications.
5. Research and consider Better Auth's capabilities and integration requirements. Clearly distinguish features supported natively by Better Auth from those requiring custom NestJS implementations.
6. Design a complete, phased implementation roadmap, including dependencies and the correct implementation order.

# Documentation Deliverables

Create all planning and technical documentation as **Markdown (`.md`) files** under:

`C:\personal-projects\nestjs-better-auth\docs`

Organize documentation into numbered folders following the intended implementation sequence.

Example:

```text
/docs
  /00-project-overview
  /01-package-installation
  /02-common-infrastructure
  /03-database-and-persistence
  /04-authentication
  /05-session-management
  /06-authorization
  /07-security
  /08-identity-management
  /09-api-integration
  /10-testing-strategy
  /11-deployment
  /12-future-enhancements
```

You may adjust, expand, or reorganize these folders based on your architectural analysis.

# Feature Planning

Cover the existing README requirements and evaluate additional production-grade capabilities, including:

- Authentication, registration, and account lifecycle
- Email verification and password recovery
- Multi-factor authentication (MFA / 2FA)
- Passkeys, OAuth 2.0, OIDC, and social authentication
- Sessions, refresh tokens, device management, and revocation
- Role-based access control (RBAC) and permissions
- Service-to-service authentication and API security
- Security events, audit logs, device/IP tracking, and abuse prevention
- Rate limiting, CSRF protection, and account lockout
- Multi-application integration and identity management
- Monitoring, observability, and production readiness

Consider additional capabilities only when they provide architectural value. Separate **MVP, production enhancements, and optional future features** to prevent unnecessary overengineering.

# Documentation Requirements

Each feature folder should contain Markdown documentation describing:

1. **Objective and scope** — what the feature accomplishes.
2. **Architecture and design** — modules, responsibilities, and interactions.
3. **Dependencies and prerequisites** — required packages, infrastructure, and preceding features.
4. **Database design** — entities, relationships, migrations, and ownership where applicable.
5. **API contracts** — planned endpoints, authentication requirements, request/response structures, and error handling.
6. **Security considerations** — threats, validation, permissions, and applicable best practices.
7. **Implementation plan** — detailed, actionable steps in the correct order.
8. **Acceptance criteria** — measurable conditions defining completion.

Include architecture diagrams using Mermaid where useful.

Create a root `/docs/README.md` serving as the documentation index, architectural overview, and master implementation roadmap.

# Expected Outcome

Deliver a **complete, organized, implementation-ready documentation system** that another AI coding agent or software engineer can follow feature by feature without needing to redesign the architecture.

Ensure all proposed features are compatible with the existing project, clearly identify assumptions and architectural decisions, and avoid duplicating functionality already provided by Better Auth.

**Important:** This is a planning and documentation task only. Do not implement features, modify application source files, or execute builds, lint checks, or tests.


# Official Documentation References

Use the following official documentation as primary technical references:

- **Better Auth:** https://better-auth.com/docs/introduction
- **NestJS:** https://docs.nestjs.com/

## Documentation Research Rules

1. Search and review the official documentation before designing each feature or architectural component.
2. Explore relevant documentation pages, APIs, plugins, configuration options, and recommended integration patterns.
3. Prioritize official documentation over third-party tutorials or outdated examples.
4. Verify compatibility between Better Auth, NestJS, PostgreSQL, and TypeORM.
5. Identify which capabilities Better Auth provides natively and which require custom NestJS implementations.
6. Include relevant official documentation links in each feature's Markdown files for future implementation reference.
7. Do not assume APIs, plugins, or features exist without verifying them against the documentation.
8. If browsing or documentation access is unavailable, explicitly document unverified assumptions rather than inventing implementation details.

**Important:** Documentation research is permitted and encouraged, but the task remains strictly architecture planning and Markdown documentation. Do not implement application code.