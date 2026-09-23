# Changelog

## Upstream 0.4 baseline

Updated the documented public baseline to IDAX Core Runtime, Shell, Ledger and osTRIS 0.4.0, and recorded IDAX Module DevKit 1.1.0 as an available public tool without adopting it into the established STIR repositories. Shell's former temporary compatibility patches are now incorporated upstream; the remaining Ledger and osTRIS patches are still explicit, reviewed public-source compatibility inputs. The initial 2026-09-23 pass left every Docker-dependent check blocked on a local Docker Desktop failure; a same-day follow-up with Docker operational completed the full battery - clean `down -v`/`build --no-cache`/`up -d`, migrations from an empty volume, runtime PostgreSQL identity and RLS, and every existing smoke/multitenant/marketplace/economic/browser E2E - with no new bugs found. See VALIDATION.md's "Public upstream 0.4 baseline" section for the complete, non-inferred evidence.

## 0.5.0-rc1

Public Beta hardening: no new product functionality on top of 0.4 - the same software made
production-deployable. A production Docker Compose overlay (`compose.production.yml` +
`Caddyfile.production`) adds automatic HTTPS, real internet-facing ports, parameterized
bootstrap-admin/site identity (no more `admin@stir.test`), and disables Swagger/OpenAPI docs on
all four Spring services (a real, previously-unnoticed gap: all of them `permitAll()`'d
`/swagger-ui/**`/`/v3/api-docs/**` publicly). `backup.py` gained retention pruning and dev/prod
awareness; `restore.py` requires typing the compose project name in production (not just `--yes`)
so a copy-pasted dev command can never wipe live data; `deploy.py` and `status.py` are new. Fixed
the ACTUAL root cause of the client-side permission-visibility gap 0.4 had worked around:
IDAX Shell's `Session.user` never carried a `permissions` field at all - committed as a public
Shell patch (`patches/idax-shell-0.5-session-permissions.patch`), not a second STIR-side
workaround. Also turned `vendor/ostris`'s accumulated, previously-unreproducible file-copy drift
into a proper patch, so a genuinely fresh clone now reproduces the real running stack. See
VALIDATION.md's 0.5 section for the full clean-build gate, security review and E2E evidence, and
the delivery report for exactly what remains blocked on external infrastructure.

## 0.4.0-SNAPSHOT

Public Pilot MVP: STIR becomes usable end to end by a non-technical person - a Home dashboard, real photo/avatar uploads to S3-compatible object storage (MinIO locally, any real provider in production, never Postgres), persistent in-app notifications for 8 negotiation/trade events, a multi-device credential lifecycle built entirely on osTRIS's existing credential/controller_credential_binding contract (add/revoke a device, self-lockout guarded, no AccountControlPolicy change), minimal content moderation (report/hide/dismiss, entirely separate from osTRIS Findings/PENALTY/RESTITUTION), fully configuration-driven instance branding, plain-language economic error messages, single-node rate limiting on especially sensitive endpoints, request-correlated logging, and a production-like Docker path (prepared, not executed against a real destination). A real, actually-executed local backup/restore round trip (Postgres + object storage, both Docker volumes destroyed and recreated from nothing) is documented in VALIDATION.md, which also records the most significant bug this phase found and fixed: `TradeService.authorize()` was relaying the FIRST-activation-time credential id to osTRIS's signature verification regardless of which device actually signed, making a second device's otherwise-correct signature always fail - the entire multi-device feature was non-functional for its actual purpose until that fix. ARCHITECTURE, DOMAIN_MODEL, PRIVACY_MODEL and README updated for the new object-storage dependency, the four new `stir` schema tables, and the device/moderation/notification domains.

## 0.3.0-SNAPSHOT

Economic Exchange MVP: Ana and Pedro can now negotiate, reach an Agreement, bind their STIR identities to explicit osTRIS participants/accounts (`MarketplaceEconomicBinding`/`ParticipantEconomicBinding`), generate an Ed25519 keypair client-side, sign a real osTRIS EXCHANGE transaction proposal STIR creates on their behalf, have osTRIS validate and commit it, and see the resulting balance change and receipt in STIR - all reconcilable by re-reading osTRIS's own authoritative status. osTRIS itself gained a minimal public discovery/provisioning/status surface (in the separate `github-public/ostris` repository) to close the gaps OSTRIS_INTEGRATION.md previously documented as blocking; no normative osTRIS semantics changed. AgreementSnapshot canonicalization moved from STIR's hand-rolled fixed-schema JSON to real RFC 8785 JCS, cross-verified between Java and JavaScript with a shared test vector. ARCHITECTURE, DOMAIN_MODEL, OSTRIS_INTEGRATION, TRANSACTION_LIFECYCLE, PRIVACY_MODEL and README updated; VALIDATION.md records a new 0.3 section covering both a full HTTP E2E (including a real credit-floor policy rejection and idempotency proof) and a real two-browser-session E2E, plus every bug found and fixed during that validation.

## 0.2.0-SNAPSHOT

Marketplace MVP documented and validated: DOMAIN_MODEL, ARCHITECTURE, TRANSACTION_LIFECYCLE, OSTRIS_INTEGRATION and IMPLEMENTATION_READINESS updated for ParticipantProfile/Offer/Negotiation/Agreement/AgreementSnapshot. VALIDATION.md records a new 0.2 section: full test/build/i18n suite, no-cache Docker rebuild, empty-volume migrations, the new HTTP and two-context-browser marketplace E2E (Ana/Pedro reach a verifiable Agreement; Carlos and a Tenant B participant are denied), and a re-check confirming the osTRIS public discovery/status gaps are unchanged.

## 0.1.0-SNAPSHOT

Public Listing foundation validated: authenticated tenant-scoped CRUD, public-source composition, separate migration/runtime identities, live A/B and PostgreSQL RLS proof, twelve-locale Shell extension and independent workspace coordination. Economic operations remain disabled.
