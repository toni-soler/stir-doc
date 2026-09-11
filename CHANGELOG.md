# Changelog

## 0.3.0-SNAPSHOT

Economic Exchange MVP: Ana and Pedro can now negotiate, reach an Agreement, bind their STIR identities to explicit osTRIS participants/accounts (`MarketplaceEconomicBinding`/`ParticipantEconomicBinding`), generate an Ed25519 keypair client-side, sign a real osTRIS EXCHANGE transaction proposal STIR creates on their behalf, have osTRIS validate and commit it, and see the resulting balance change and receipt in STIR - all reconcilable by re-reading osTRIS's own authoritative status. osTRIS itself gained a minimal public discovery/provisioning/status surface (in the separate `github-public/ostris` repository) to close the gaps OSTRIS_INTEGRATION.md previously documented as blocking; no normative osTRIS semantics changed. AgreementSnapshot canonicalization moved from STIR's hand-rolled fixed-schema JSON to real RFC 8785 JCS, cross-verified between Java and JavaScript with a shared test vector. ARCHITECTURE, DOMAIN_MODEL, OSTRIS_INTEGRATION, TRANSACTION_LIFECYCLE, PRIVACY_MODEL and README updated; VALIDATION.md records a new 0.3 section covering both a full HTTP E2E (including a real credit-floor policy rejection and idempotency proof) and a real two-browser-session E2E, plus every bug found and fixed during that validation.

## 0.2.0-SNAPSHOT

Marketplace MVP documented and validated: DOMAIN_MODEL, ARCHITECTURE, TRANSACTION_LIFECYCLE, OSTRIS_INTEGRATION and IMPLEMENTATION_READINESS updated for ParticipantProfile/Offer/Negotiation/Agreement/AgreementSnapshot. VALIDATION.md records a new 0.2 section: full test/build/i18n suite, no-cache Docker rebuild, empty-volume migrations, the new HTTP and two-context-browser marketplace E2E (Ana/Pedro reach a verifiable Agreement; Carlos and a Tenant B participant are denied), and a re-check confirming the osTRIS public discovery/status gaps are unchanged.

## 0.1.0-SNAPSHOT

Public Listing foundation validated: authenticated tenant-scoped CRUD, public-source composition, separate migration/runtime identities, live A/B and PostgreSQL RLS proof, twelve-locale Shell extension and independent workspace coordination. Economic operations remain disabled.
