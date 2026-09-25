# Community references v0.1 — change and validation record

All paths below are relative to `stir/`. Each named child remains its own Git
repository. Backend, frontend and docs use local branch
`codex/community-value-references`; deployment/E2E uses
`codex/community-value-references-e2e` because its original branch name already
existed with a separate commit. No osTRIS Core, IDAX Core, .NET project or
private book repository was changed.

## Exact changed files

**stir-backend**

```text
config/permission-catalog.json
scripts/generate-permissions.py
src/main/java/org/stir/reference/EvidenceAnalysis.java
src/main/java/org/stir/reference/ReferenceAcceptanceAdapter.java
src/main/java/org/stir/reference/ReferenceController.java
src/main/java/org/stir/reference/ReferenceService.java
src/main/java/org/stir/listing/Listing.java
src/main/java/org/stir/listing/ListingRequest.java
src/main/java/org/stir/listing/ListingService.java
src/main/java/org/stir/listing/ListingView.java
src/main/java/org/stir/negotiation/CounterRequest.java
src/main/java/org/stir/negotiation/Negotiation.java
src/main/java/org/stir/negotiation/NegotiationActionRequest.java
src/main/java/org/stir/negotiation/NegotiationController.java
src/main/java/org/stir/negotiation/NegotiationDetail.java
src/main/java/org/stir/negotiation/NegotiationService.java
src/main/java/org/stir/negotiation/Offer.java
src/main/java/org/stir/negotiation/OfferRequest.java
src/main/resources/db/migration-stir/V5__community_value_references.sql
src/main/resources/generated/stir/permission-catalog.generated.json
src/test/java/org/stir/negotiation/NegotiationServiceTest.java
src/test/java/org/stir/reference/EvidenceAnalysisTest.java
src/test/java/org/stir/reference/ReferencePostgresTest.java
```

`config/permission-catalog.json` is the new source for the permission catalog;
`scripts/generate-permissions.py` is the generator and the JSON under
`resources/generated/` is its regenerated output. No other generator, template
or generated file changed.

**stir-frontend**

```text
src/agreement.jsx
src/api.js
src/extension.jsx
src/listing.jsx
src/locales.json
src/negotiation.jsx
src/reference-comparison.js
src/references.jsx
tests/references.test.mjs
```

**stir-main**

```text
scripts/community_references_e2e.py
scripts/community_references_browser.py
```

**stir-doc**

```text
COMMUNITY_VALUE_REFERENCES.md
VALIDATION_COMMUNITY_VALUE_REFERENCES.md
```

## Gate results

| Gate | Evidence |
|---|---|
| Backend verify | `mvn verify`: 100 tests, 0 failures/errors, including Testcontainers PostgreSQL |
| Empty Flyway | PostgreSQL 17 Testcontainers applied V1–V5 from zero; live Docker module migration applied V5 |
| RLS and immutability | Real `idax_app`/`idax_admin` tenant isolation, FORCE RLS catalog check, SQL immutable trigger test |
| Frontend | `npm test`: 27 tests, 0 failures; `npm run build`: pass |
| i18n | `npm run i18n:validate`: 12 locales with identical key sets, pass |
| HTTP E2E | `community_references_e2e.py`: published v1/v2, three free agreement amounts, consent, private access, tenant A/B, unchanged historical context, digests, daily cutoff |
| Browser E2E | `community_references_browser.py`: two isolated sessions, v1 publication, linked listing, human explanation, below/above negotiation, Agreement, v2 publication and historical v1 |
| Docker | `docker compose build --no-cache stir stir-ui`: pass; frontend image rebuilt without cache after final UI changes |
| Restart | Running STIR containers recreated; aggregate digest of persisted reference snapshots unchanged |

Contract review verified that osTRIS EXCHANGE already receives only the existing
AgreementSnapshot digest as `contractualMetadataDigest` and the accepted amount.
No Java–.NET or .NET Framework contract applies to STIR's new local context.
The new reference context digest is a separate STIR audit artifact and is not
claimed to be anchored in osTRIS by this version.

The local browser creates disposable development fixture tenants and leaves its
screenshots under `stir-main/.local/`; these files are ignored by Git. Live
Docker backend/frontend replacements are stopped after validation in accordance
with the workspace working-mode instruction.
