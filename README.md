# STIR

**Sistema Transparente de Intercambio de Recursos** is an open source marketplace built on osTRIS. STIR application source is Apache-2.0.

`stir.es` is a future public reference instance. Anyone can deploy an independent instance with its own hostname, users, database, secrets, tenants and explicitly configured economic communities. No runtime logic depends on that hostname.

STIR uses the public IDAX Open Core, IDAX Shell, IDAX Ledger and osTRIS 0.4 baselines. Core is publicly obtainable but separately licensed and is not open source. The entire dependency stack must not be described as Apache-2.0 or reproducible from source: Core's published binary is an explicitly accepted build input.

Start with [Architecture](ARCHITECTURE.md), [Domain](DOMAIN_MODEL.md), [Integration](OSTRIS_INTEGRATION.md), [Public boundary](PUBLIC_SOFTWARE_BOUNDARY.md) and [Readiness](IMPLEMENTATION_READINESS.md). Run the local instance from sibling `stir-main`.

Version: `0.5.0-rc1`. 0.5 adds no product functionality on top of 0.4 (photos, notifications, multi-device credential lifecycle, minimal content moderation, configuration-driven instance branding - see [Domain](DOMAIN_MODEL.md)'s "Public pilot additions" section); it hardens the same software into a production-deployable public beta - see [Validation](VALIDATION.md)'s 0.5 section for the production topology, security gate and clean-build evidence. Economic exchange (Agreement -> real osTRIS EXCHANGE -> commit) is implemented for development/validation; see [Integration](OSTRIS_INTEGRATION.md) and [Transaction lifecycle](TRANSACTION_LIFECYCLE.md).
