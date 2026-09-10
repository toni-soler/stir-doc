# Validation evidence

Validation performed 2026-09-10–11. Commands run from the named repository. No passwords, tokens or private keys are included. PASS denotes the development foundation, not production readiness.

| Check | Command / mechanism | Status | Observed result |
|---|---|---|---|
| Backend | stir-backend: `mvn -s .mvn/public-settings.xml clean verify` | PASS | BUILD SUCCESS; 9 tests, 0 failures/errors/skipped. Seven Listing tests, one PostgreSQL/Testcontainers migration/RLS test, one validated-service-identity rejection test. |
| Frontend clean install | stir-frontend: `npm ci` | PASS | Clean public npm lockfile install. |
| Frontend tests | `npm test` | PASS | 4 tests, 0 failures. |
| Translations | `npm run i18n:validate` | PASS | 12 locales, 51 keys each. |
| Frontend build | `npm run build` | PASS | esbuild extension produced successfully. |
| Workspace | root: `python scripts/validate-workspace.py`; `git ls-files` | PASS | Five independent Git roots; relative paths/tasks; eight coordinator files; no child files or gitlinks. No duplicate workspace in stir-main. |
| Public origin/pins | Compare vendor HEAD/origin to upstream.lock.json; reviewed patch guard | PASS | Four public origins/commits match. Unexpected vendor source rejected; reviewed patches accepted idempotently. |
| Secret hygiene | Compare staged files against current local secret values; Git ignore checks | PASS | No current secret in staged content. Existing regenerated secrets preserved for final reset. |
| Shell Java | public Shell fix: `mvn -s ../../stir-backend/.mvn/public-settings.xml verify` | PASS | 6 tests, BUILD SUCCESS. |
| Shell frontend | public Shell fix/frontend: `npm ci`, `npm test`, `npm run build` | PASS | 4 tests and Vite build passed. |
| Shell patch identity | Compare shipped patch with Git diff base..afb3dce | PASS | Exact match to independent generic public-source commit. |
| Final clean reset | stir-main: `docker compose down -v` | PASS | Previous development volume and containers removed; secrets retained. |
| Final no-cache build | `docker compose build --no-cache` | PASS | All source images built without cache; no private registry/artifact or host-built JAR. |
| Final empty-volume migrations | `docker compose up -d`; migration job logs | PASS | Empty volume: Core 91, Shell 1, STIR 1, Ledger 3, osTRIS 4 migrations. Migration and role-provision jobs exited 0. |
| Health checks | `docker compose ps --format json` | PASS | PostgreSQL, Shell, STIR, Ledger, osTRIS and proxy healthy; all three UI containers running. |
| Final runtime identity/RLS | `python scripts/multitenant.py` | PASS | Actual JDBC sessions: idax_backend; rolsuper=false; rolbypassrls=false; no schema/database CREATE. Forced listing policy isolates both idax_app/idax_admin; no tenant sees zero rows. |
| Final Listing HTTP | `python scripts/smoke.py` | PASS | Login; absent/malformed/tampered-signature JWT 401; catalogs; create/read/list/filter/update; stale 409; close; repeated close idempotent; CLOSED edit 409; DELETE 405; bounded pagination 400. |
| Final A/B HTTP | `python scripts/multitenant.py` | PASS | Both ordinary owners read/update; foreign UUID read/update/close 404; foreign tenant paths and contradictory headers denied; combined filters isolated; owner forgery rejected. |
| Final public boundary | `python scripts/audit-public.py` | PASS | 99 owned text files across five repositories; only two explanatory/scanner-pattern matches. No prohibited dependency, home path, binary or key. |
| Browser | `python scripts/browser-smoke.py` with requirements-browser.txt and Edge | PASS | Edge: login, tenant selection, Shell module load, WANTED create/list/filter/edit/close, CLOSED query, navigation and deep-link reload. No JavaScript page errors. |
| Shutdown | `docker compose down`; `docker compose ps -a` | PASS | Agent-started services removed after verification; development volume and current secrets retained. |
| Economic transaction | No command: deliberately unimplemented | NOT RUN | Outside Foundation; public osTRIS discovery/status/provisioning/signing-payload gaps documented in OSTRIS_INTEGRATION.md. |

## Runtime proof semantics

Core's public tenant_create database capability provisions fixture tenants because Shell has no public tenant-creation HTTP route. User, role, permission and role-assignment fixtures use public Shell HTTP only. No direct Core table inserts/updates or economic fixtures are used. Test passwords remain in memory; local evidence files contain fixture identifiers only.

The A/B test checks both tenants symmetrically, own update, foreign UUID read/update/close, foreign tenant path, contradictory X-Tenant, and combined text/category/resourceKind/status filters. Runtime SQL records session identity, role attributes/memberships, enabled/forced RLS and policy expressions. Both idax_app and idax_admin see only the transaction-local tenant; no tenant context sees zero rows. This is executed against the actual deployed database, independently of Testcontainers.

## Earlier failures resolved before the final clean run

The initial direct TokenValidator adapter allowed login but failed authenticated catalogs (403); delegating Core JwtAuthFilter fixed this through the public authentication contract. Initial A/B fixture setup omitted separate role-permission/user-role calls; the corrected fixture uses those actual public routes and passes. A previous Docker attempt hit transient registry/Alpine network failures; unnecessary curl installation was removed in favor of wget already in the runtime image. An audit false positive on the public /roles/users/ endpoint was corrected to distinguish an API path from a filesystem home path. The browser test initially used exact label matching on wrapped selects; matching the actual accessible label fixed the test without changing application UI.

The public-source Shell adapter remains explicitly temporary. Its fix belongs to idax-shell, commit afb3dce1ef7d6f722f18f804bb1f70f48687168c, branch codex/public-extension-contract. No push, tag, release or production deployment occurred.

## Visual observation

The browser flow is functional and translated. Screenshots also show inherited Shell styling needing later visual cleanup: a missing Shell logo asset and low contrast in the module introduction/pagination on the light Shell background. No UI redesign was performed in this validation-only pass. These are cosmetic limitations, not failures of authentication, tenant isolation or the Listing lifecycle.
