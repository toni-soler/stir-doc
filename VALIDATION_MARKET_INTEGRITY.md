# Market integrity and Seven Keys: change record

All four independent STIR repositories use the local branch
`codex/market-integrity-seven-keys`. No osTRIS, IDAX Core, .NET or private book
repository was changed. No generator or template was changed and no generated
file was edited.

## Exact changed files

**stir-backend**

```text
src/main/java/org/stir/reference/EvidenceAnalysis.java
src/main/java/org/stir/reference/ReferenceController.java
src/main/java/org/stir/reference/ReferenceService.java
src/main/java/org/stir/reference/MarketIntegrityController.java
src/main/java/org/stir/reference/MarketIntegrityService.java
src/main/java/org/stir/reference/SevenKeysController.java
src/main/java/org/stir/reference/SevenKeysCrypto.java
src/main/java/org/stir/reference/SevenKeysService.java
src/main/resources/db/migration-stir/V6__market_integrity_governance.sql
src/main/resources/db/migration-stir/V7__enforce_seven_seats.sql
src/main/resources/db/migration-stir/V8__integrity_case_sequence.sql
src/main/resources/db/migration-stir/V9__restrict_constitutional_mutation_role.sql
src/test/java/org/stir/reference/EvidenceAnalysisTest.java
src/test/java/org/stir/reference/ReferencePostgresTest.java
src/test/java/org/stir/reference/SevenKeysCryptoTest.java
src/test/java/org/stir/reference/SevenKeysPostgresTest.java
```

**stir-frontend**

```text
src/api.js
src/locales.json
src/references.jsx
```

**stir-main**

```text
scripts/market_integrity_e2e.py
```

**stir-doc**

```text
MARKET_INTEGRITY.md
SEVEN_KEYS_GOVERNANCE.md
CREDENTIAL_RECOVERY.md
GOVERNANCE_CAPTURE_THREAT_MODEL.md
VALIDATION_MARKET_INTEGRITY.md
```

## Gates

| Gate | Result and evidence |
|---|---|
| Backend verify | PASS: `mvn verify`, 116 tests, no failures; existing coverage retained |
| Empty Flyway | PASS: PostgreSQL 17 Testcontainers applies V1–V9 from zero; live local Docker migration applied V6–V9 |
| PostgreSQL/RLS | PASS: FORCE RLS and tenant A/B, immutable history, deferred seven-seat constraint, admin DB role cannot rotate seats |
| Market integrity | PASS for account/pair/time/sensitivity signals, immutable raw observations, reviewed FINAL/DISMISSED eligibility and reproducible reason codes; no verified cross-account identity clusters yet |
| Seven Keys | PASS for one-time 7-seat bootstrap, canonical Ed25519 signatures, 6/7 rejection, 7/7 amendment and replay resistance |
| Guardian not superadmin | PASS: one suspension, second denied, 5/7 removal possible even with a seat frozen; Guardian signature never substitutes for a seat |
| Recovery fail-closed | PASS for same-controller Guardian + 6/6 + new-key possession; old key revoked. Claimed controller-replacement finality is denied |
| Governance audit reconstruction | PASS: event sequence/JCS/SHA-256 predecessor chain recomputed from PostgreSQL; partial proposal and seat signatures retained |
| Frontend | PASS: 27 tests, build; `npm run i18n:validate` passes 12 locales with 287 keys each |
| HTTP E2E | PASS: `market_integrity_e2e.py` with real Ed25519 and PostgreSQL-backed REST; existing `community_references_e2e.py` regression passed |
| Browser E2E | PASS: existing two-session `community_references_browser.py` flow after backend/UI change, no page errors |
| Clean Docker | PASS: `docker compose build --no-cache stir stir-ui`; final backend rebuild includes V9 |

The Seven Keys HTTP E2E covers permitted operational policy adjustment,
protected mutation rejection, 6/7 and 7/7, suspension, freeze, unilateral
replacement rejection, 5/6 and 6/6 recovery, key history, Guardian removal,
redacted public audit and cross-tenant denial. PostgreSQL tests cover exact
cryptographic payload changes, unknown/revoked/suspended signers, case
transitions, source retention and database privileges. The original Agreement
and osTRIS EXCHANGE contracts are unchanged. No Java–.NET contract applies.

**Not a PASS:** controller replacement cannot activate. STIR has no trustworthy
verified FINAL resolution/appeal proof from osTRIS; see
`CREDENTIAL_RECOVERY.md`. Likewise related-account clusters require a lawful
private identity-continuity adapter. These are explicit gaps, not silently
simulated successes.

The HTTP/browser fixtures create disposable local tenants only. Agent-started
`stir` and `stir-ui` containers are stopped after verification; preexisting
PostgreSQL, osTRIS, shell, proxy and other services are left alone.
