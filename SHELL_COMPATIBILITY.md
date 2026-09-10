# Shell compatibility and upstream work

**TEMPORARY PUBLIC-SOURCE COMPATIBILITY ADAPTER**

STIR pins public Shell 0.3.0 and applies the reviewed patch shipped in stir-main/patches. This is public source and reproducible, not a private dependency. It is temporary, not the permanent extension contract.

An independent worktree/branch `codex/public-extension-contract` contains local commit `afb3dce1ef7d6f722f18f804bb1f70f48687168c` for the platform fix against public Shell commit a697c946e72feffcd20598ab61eada48d060c99a. No push, release or change to tag 0.3.0 occurs. A future published Shell version containing this fix is required to remove the adapter; no unassigned version number is promised.

Generic behavior: manifest routes are validated and unique/nonoverlapping; reserved API/system roots are forbidden. GET/HEAD SPA handlers exist only at declared prefixes. Static assets retain their handlers and protected APIs retain authentication. Frontend matches the manifest route rather than a list of product IDs. SDK exposes activeTenantId; a tenant change remounts the module and locale changes refresh SDK translation context.

The same public branch corrects the local initializer's tenant creation to use Core's public tenant_create capability. Core 0.3 migration V87 intentionally revoked broad tenant INSERT; runtime must not regain it just to run development bootstrap.

Ledger/osTRIS public custom Flyway beans have no external-migrations switch. Tiny compatibility patches add idax.module.migrations.enabled (default true). Composition sets it false after one-shot migrations. This changes no economic or evidence behavior and should be contributed to those public modules separately. No runtime has bootstrap database credentials.

The initializer checks upstream commit, exact origin, tracked diff against the shipped patch and unexpected untracked source. It never silently accepts unrelated vendor edits.

The change belongs to **idax-shell**, not STIR. Its independent public-origin checkout passed 6 Java tests and 4 frontend tests plus the Vite build. The shipped patch is the exact diff of that commit against the pinned base. STIR consumes it only until an upstream public release provides this generic contract.
