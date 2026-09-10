# Implementation readiness

Foundation validation passed; this is a development foundation, not a production release. Listing vertical does not invoke economic operations. VALIDATION.md records executed checks; unexecuted checks are never PASS.

Future economic blockers: public osTRIS discovery/read/status contracts, credential onboarding and canonical signing payload retrieval. Snapshot schema and cross-language vectors remain pending. Attachments, anonymous publication, moderation and production deployment are outside 0.1.

Shell 0.3 supplies React/router/i18n/authenticated HTTP but not CRUD or active tenant in its SDK, and hardcodes two module hosts. stir-main carries a pinned-source adapter. Ledger/osTRIS Dockerfiles reference stale 0.2.0 filenames; composition-owned Dockerfiles build public 0.3 sources. Published upstream commits are unchanged. The independent idax-shell fix and reviewed compatibility patches are documented in SHELL_COMPATIBILITY.md.
