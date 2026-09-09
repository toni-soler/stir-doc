# Public 0.3 discovery

Anonymous git ls-remote verified these toni-soler GitHub v0.3.0 commits on 2026-09-10:

| Repository | Commit |
|---|---|
| idax-core-runtime | 265b4fd52fc5d0742bb4e03ca752878f75e13c8e |
| idax-shell | a697c946e72feffcd20598ab61eada48d060c99a |
| idax-ledger | 31e56851f86ec3b327e6e6c2aec3f1590f6c00fc |
| ostris | d92aa1f605884ebdcd8e97fff46f4067b0416bcc |

Maven: es.idynamicsax.idax:idax-core:0.3.0 from https://toni-soler.github.io/idax-core-runtime/maven2. Modules use Boot parent 3.4.4, Java 21, Web/Data JPA/Validation/Actuator, Flyway PostgreSQL, springdoc 2.8.9, JUnit and Testcontainers.

Frontend: React 18.3.1 JavaScript JSX, esbuild 0.25.12; Shell uses Vite. SDK window.__IDAX_MODULE_SDK__, registry window.__IDAX_MODULE_EXTENSIONS__. Manifest schemaVersion 1, extensions array, configured through EXTENSION_MANIFEST.

Auth: Core TokenValidator, validated CurrentUser, TenantContextFilter, DbSessionContextService and RlsTransactionAspect. Permission catalog lifecycle uses ModulePermissionCatalogDescriptor and PermissionService.

Database: PostgreSQL 17, per-module Flyway schema/history, Core migrations first. RLS uses app.tenant_id and roles idax_app/idax_admin. Health: /actuator/health/readiness. Docker: source multi-stage Java 21, nginx frontend, Caddy proxy. STIR replaces prepublication local Maven proxy settings with public resolution.

osTRIS consumes Core binary, not source reactor. Ledger integration is HTTP proof outbox, disabled by default. Docs are Markdown with architecture, changelog, specification and ADRs. Actual routes are in OSTRIS_INTEGRATION.md.

No legacy prototype needed or read. Internal DevKit contract was read as workstation guidance only, not used as a build input.
