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

## 0.2 Marketplace MVP

Validation performed 2026-09-11. Same conventions as above: commands run from the named repository, no secrets included, PASS denotes the development foundation.

| Check | Command / mechanism | Status | Observed result |
|---|---|---|---|
| Backend | stir-backend: `mvn -s .mvn/public-settings.xml clean verify` (equivalent: `mvn -o test`) | PASS | 34 tests, 0 failures/errors/skipped: 7 Listing service, 1 Listing RLS/migration (now covers all five 0.2 tables), 1 personal-endpoint auth, 16 Negotiation service (state machine, party auth, concurrency), 5 CanonicalJson test vectors, 3 ParticipantProfile service. |
| Frontend clean install | stir-frontend: `npm ci` | PASS | Clean install after the 0.1→0.2 lockfile version bump. |
| Frontend tests | `npm test` | PASS | 8 tests, 0 failures (4 new: offer payload normalization, participant/negotiation/agreement API paths). |
| Translations | `npm run i18n:validate` | PASS | 12 locales, 97 keys each (46 new keys for profile/offer/negotiation/agreement screens). |
| Frontend build | `npm run build` | PASS | esbuild extension bundle, now including profile/listing/negotiation/agreement screens. |
| Workspace | root: `python scripts/validate-workspace.py` | PASS | Unchanged: five independent Git roots, no child files/gitlinks. |
| Public boundary | `python scripts/audit-public.py` | PASS | 137 owned text files across five repositories; 2 explanatory matches reviewed; no private dependency, path, key or STIR remote. |
| Docker no-cache build | stir-main: `docker compose down -v`; `docker compose build --no-cache` | PASS | All composed images (Shell, STIR, Ledger, osTRIS and their frontends) rebuilt from source with no cache; stir-backend Maven deps and stir-frontend npm deps resolved from public repositories only; stir-frontend's own `npm run i18n:validate && npm run build` ran inside the image build and passed. |
| Empty-volume migrations | `docker compose up -d`; migration job logs | PASS | Empty volume, repeated on the no-cache images: Core (unchanged), Shell 1, **STIR 2** (V1 listings + V2 marketplace), Ledger 3, osTRIS 4. Migration and role-provision jobs exited 0. |
| Health checks | `docker compose ps` | PASS | PostgreSQL, Shell, STIR, Ledger, osTRIS and proxy healthy; all three UI containers running. |
| Marketplace E2E | `python scripts/marketplace_e2e.py` | PASS | Ana and Pedro (Tenant A) set up ParticipantProfiles; Ana publishes an OFFER listing; Pedro discovers it (response includes Ana's displayName); Pedro proposes; Ana counters (history preserved, previous offer SUPERSEDED); Pedro accepts; Agreement + AgreementSnapshot created (economicPhase=AWAITING_ECONOMIC_EXECUTION); both parties read back an identical 64-hex-char digest. Carlos (same tenant, uninvolved) reads the public Listing but gets 404 on the negotiation and agreement and cannot decline. A Tenant B participant is denied on all three (400/403/404 depending on where the tenant check lands, matching the existing Listing precedent for foreign-tenant paths). Re-run after the no-cache rebuild with the same result. |
| Marketplace browser E2E | `scripts/marketplace_browser_smoke.py` (two independent Playwright browser contexts = two real sessions) | PASS | Ana logs in, sets her profile, publishes "20kg tomatoes"; Pedro logs in (separate context/session), finds it via search, opens the detail page, submits an offer; Ana opens My negotiations, sees Pedro's offer, submits a counteroffer; Pedro reloads, sees the counteroffer, accepts; lands on a real rendered Agreement page showing the AWAITING_ECONOMIC_EXECUTION phase. No JavaScript page errors in either session. Screenshot: `.local/browser-agreement.png`. |
| Browser (existing) | `python scripts/browser-smoke.py` | PASS | Unaffected by 0.2: login, tenant selection, Shell module load, WANTED create/list/filter/edit/close, CLOSED query, navigation and deep-link reload; no JavaScript page errors. |
| A/B HTTP (existing) | `python scripts/multitenant.py` | PASS | Unaffected by 0.2; runtime SQL also confirms forced RLS policies now exist and are correctly tenant-scoped on `negotiation`, `offer`, `agreement`, `agreement_snapshot` and `participant_profile`, not just `listing`. |
| Listing HTTP (existing) | `python scripts/smoke.py` | PASS | Unaffected by 0.2. |
| osTRIS gap re-check | Inspected `vendor/ostris/backend/src/main/java/.../api/*Controller.java` | Confirmed unchanged | Only `TransactionController` (POST proposals/authorizations/governance-authorizations/commit) and `IdentityContinuityController` exist; still no discovery/read, participant/community/unit lookup, or GET status endpoint. OSTRIS_INTEGRATION.md's documented gaps stand; 0.2 does not work around them (see "osTRIS blockers for next MVP" in the delivery summary). |
| Economic transaction | No command: deliberately unimplemented | NOT RUN | Same as 0.1: outside this MVP: `Agreement.economicPhase` stays `AWAITING_ECONOMIC_EXECUTION`. |

Runtime proof semantics, fixture provisioning and the "earlier failures resolved" history above are unchanged for 0.2 - `marketplace_e2e.py`/`marketplace_browser_smoke.py` reuse the exact same isolated fixture pattern (`idax_core.tenant_create` + public Shell user/role/permission HTTP endpoints, never a direct Core table write) as `multitenant.py`.

## 0.3 Economic Exchange MVP

Validation performed 2026-09-11. Same conventions as above: commands run from the named repository, no secrets included, PASS denotes the development foundation. osTRIS changes were made and validated in the separate `github-public/ostris` repository (STIR's `vendor/ostris` is a read-only reproducibility copy, synced from there for this Docker build - never edited directly).

| Check | Command / mechanism | Status | Observed result |
|---|---|---|---|
| osTRIS backend | `mvn -o test` (github-public/ostris/backend) | PASS | 245 tests across 18 classes, 0 failures/errors/skipped. Includes 6 new `ProvisioningAndDiscoveryPostgresTest` cases (provision -> discover -> propose -> real-Ed25519 sign -> commit -> verify balance; credit-floor rejection leaves balance/journal untouched; discovery fails closed on unknown IDs and is tenant-scoped; provisioning rejects unknown community/unit, duplicate unit code, duplicate credential key, invalid unit code/credit floor/public key) and updated permission-catalog/security tests for the 4 new permission codes (10 total). No normative osTRIS test was weakened. |
| Backend | stir-backend: `mvn -s .mvn/public-settings.xml clean verify` (equivalent: `mvn -o test`) | PASS | 52 tests, 0 failures/errors/skipped: 18 new (6 `EconomicActivationServiceTest`, 10 `TradeServiceTest`, 2 more `CanonicalJsonTest` vectors), `ListingRlsTest` now also covers the three 0.3 economic tables, all 0.2 suites unaffected. |
| Frontend clean install | stir-frontend: `npm ci` | PASS | Clean install after the 0.2->0.3 lockfile version bump (adds `canonicalize`). |
| Frontend tests | `npm test` | PASS | 19 tests, 0 failures (11 new: 5 `signer.test.mjs`, 5 `canonical-json.test.mjs` shared-vector, 1 confirming a 200-with-empty-body response resolves to `null` not `{}`). |
| Translations | `npm run i18n:validate` | PASS | 12 locales, 132 keys each (35 new keys for the economic activation/trade/receipt screens). |
| Frontend build | `npm run build` | PASS | esbuild extension bundle, now including the economic activation and trade-status screens; zero warnings (a duplicate-key collision from a hastily-named new locale key was caught and fixed before this run - see "Earlier failures resolved" below). |
| Workspace | root: `python scripts/validate-workspace.py` | PASS | Unchanged: five independent Git roots, no child files/gitlinks. |
| Public boundary | `python scripts/audit-public.py` | PASS | 160 owned text files across five repositories; 2 explanatory matches reviewed; no private dependency, path, key or STIR remote. |
| Docker build | stir-main: `docker compose down -v`; `docker compose build` | PASS | All composed images (Shell, STIR, Ledger, osTRIS and their frontends, including the synced `vendor/ostris` with the new discovery/provisioning code) rebuilt; stir-frontend's own `npm run i18n:validate && npm run build` ran inside the image build and passed. |
| Empty-volume migrations | `docker compose up -d`; migration job logs | PASS | Empty volume: STIR now applies 3 migrations (V1 listings + V2 marketplace + V3 economic exchange). Migration and role-provision jobs exited 0. |
| Health checks | `docker compose ps` | PASS | PostgreSQL, Shell, STIR, Ledger, osTRIS and proxy healthy; all three UI containers running. |
| Economic exchange HTTP E2E | `python scripts/economic_exchange_e2e.py` | PASS | Ana and Pedro activate economic exchange with their own client-generated Ed25519 keys. OFFER: Ana offers "20kg tomatoes", Pedro proposes 1000, Ana counters 900, Pedro accepts (payerUserId=Pedro, payeeUserId=Ana); both sign; osTRIS commits; Ana's balance +900, Pedro's -900. Reconciliation: `sync()` reports the same committedSequence/protocolDigest as the commit receipt. Idempotency: repeated activate/commit calls create no second Trade or journal entry. WANTED: Ana (requester) pays Pedro (provider) 500, confirming direction is not hardcoded to OFFER. Policy rejection: a 500000 OFFER breaches Pedro's -100000 credit floor; osTRIS's commit call fails (422), the Agreement still exists, the Trade is REJECTED, neither balance moved, no journal entry was written - STIR never overrode osTRIS's decision. |
| Economic exchange browser E2E | `python scripts/economic_browser_smoke.py` | PASS | Two real, independent browser sessions (Ana, Pedro), each generating its own Ed25519 keypair via WebCrypto and keeping it in that session's own IndexedDB (never shared, never sent to either server). Ana bootstraps the marketplace's economic community/unit and activates; Pedro activates; Ana publishes a WANTED listing, Pedro proposes, Ana accepts directly; Ana starts the exchange, both sign through the real rendered UI, Pedro commits; both sessions reach a visible COMMITTED receipt (committedSequence/protocolDigest/committedAt) with no JavaScript page errors. |
| Browser (existing) | `python scripts/browser-smoke.py` | PASS | Unaffected by 0.3. |
| Marketplace browser E2E (existing) | `python scripts/marketplace_browser_smoke.py` | PASS | Its accepted-Agreement assertion was updated from the old always-AWAITING_ECONOMIC_EXECUTION label to "Sin intercambio económico" (NOT_APPLICABLE) - this negotiation never sets an amount, so 0.3's new payer/payee-direction branching correctly resolves it as a free/non-monetary exchange; this is the intended new behavior, not a regression. |
| A/B HTTP (existing) | `python scripts/multitenant.py` | PASS | Unaffected by 0.3; runtime SQL also confirms forced RLS policies now exist and are correctly tenant-scoped on `marketplace_economic_binding`, `participant_economic_binding` and `trade`. |
| Listing HTTP / Marketplace E2E (existing) | `python scripts/smoke.py`, `python scripts/marketplace_e2e.py` | PASS | Unaffected by 0.3. |

## Bugs found and fixed during 0.3 validation

None of these were pre-existing 0.2 regressions; all were introduced by 0.3 work and caught by validation before this PASS, not after:

- `agreement_payer_not_payee` CHECK constraint used `payer_user_id IS DISTINCT FROM payee_user_id`, but SQL treats two NULLs as NOT distinct - this rejected the legitimate NOT_APPLICABLE case (both NULL) along with the real bug it was meant to catch. Fixed to `payer_user_id IS NULL OR payer_user_id IS DISTINCT FROM payee_user_id`.
- `OstrisClient.post()` never called a terminal method on Spring's `RestClient` `ResponseSpec` for `Void`-typed (bodiless) responses - so `retrieve()`'s default error handling never ran and a 4xx/5xx from osTRIS's authorization endpoint passed silently as "success". Fixed to always call a terminal method (`toBodilessEntity()`/`toEntity()`).
- `TradeService.commit()`'s catch block wrote the REJECTED status and then rethrew, inside a class-level `@Transactional` method - Spring rolled the whole transaction back on the way out, silently undoing the REJECTED write. Fixed by recording it through a separate `@Transactional(REQUIRES_NEW)` bean (`TradeRejectionRecorder`), so the write survives.
- `economic.jsx` treated `EconomicActivationService.marketplace()`'s "not yet bound" response as 404, but the backend actually returns 409 - the UI showed "Loading..." forever instead of the bootstrap form. Fixed the status check.
- `api.js`'s generic request helper fell back to `{}` when a response body failed to parse as JSON, which is exactly what a 200-with-empty-body response does (Spring writes nothing for a `null` controller return value) - `TradeStatus`'s `trade === null` check never matched, so the "not yet activated" UI never showed. Fixed to parse an empty body as `null`.

## Earlier failures resolved before this PASS

A new locale key named `community` (the osTRIS economic community) collided with the pre-existing `community` catalog-code translation key; caught via `npm run build`'s esbuild duplicate-key warnings (not by the user), renamed to `economicCommunity`. Two browser-script bugs were self-caught before their first real run: an ambiguous `get_by_text("Confirmado")` assertion (both the Agreement page's economicPhase badge and TradeStatus's own badge can read "Confirmado") fixed by asserting on the unique receipt-only "Secuencia de la comunidad" text; and every `withNav`-wrapped page (listing/negotiation/agreement/profile detail) renders only a single "← Marketplace" back button, never the full tab bar - navigation steps that follow such a page now go through that back button first. `get_by_label` was unreliable for a `<select>` wrapped in a `<label>` in this Chromium/Playwright combination (it worked for `<input>`); switched to `get_by_role("combobox", ...)`, the more idiomatic Playwright pattern for `<select>` elements anyway.
