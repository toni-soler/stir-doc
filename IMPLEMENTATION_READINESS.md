# Implementation readiness

Foundation validation passed; 0.2 adds a full marketplace MVP (ParticipantProfile, Offer, Negotiation, Agreement, AgreementSnapshot) on top of it. Neither is a production release. No STIR code path invokes an osTRIS economic operation; Agreement.economicPhase stays AWAITING_ECONOMIC_EXECUTION. VALIDATION.md records executed checks; unexecuted checks are never PASS.

Future economic blockers (unchanged from 0.1, re-checked for 0.2): public osTRIS discovery/read/status contracts, credential onboarding and canonical signing payload retrieval. AgreementSnapshot's own schema and canonicalization are now implemented and test-vectored (OSTRIS_INTEGRATION.md); cross-language vectors against a real osTRIS EXCHANGE call remain future work once those contracts exist. Attachments, anonymous publication, moderation and production deployment remain outside 0.2.

Shell 0.3 supplies React/router/i18n/authenticated HTTP but not CRUD or active tenant in its SDK, and hardcodes two module hosts. stir-main carries a pinned-source adapter. Ledger/osTRIS Dockerfiles reference stale 0.2.0 filenames; composition-owned Dockerfiles build public 0.3 sources. Published upstream commits are unchanged. The independent idax-shell fix and reviewed compatibility patches are documented in SHELL_COMPATIBILITY.md.
