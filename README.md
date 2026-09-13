# STIR

**Sistema Transparente de Intercambio de Recursos** is an open source marketplace built on osTRIS. STIR application source is Apache-2.0.

`stir.es` is a future public reference instance. Anyone can deploy an independent instance with its own hostname, users, database, secrets, tenants and explicitly configured economic communities. No runtime logic depends on that hostname.

STIR uses public IDAX Open Core 0.3 runtime artifacts and the public IDAX Shell. Core is publicly obtainable but separately licensed and is not open source. The entire dependency stack must not be described as Apache-2.0 or reproducible from source: Core's published binary is an explicitly accepted build input.

Start with [Architecture](ARCHITECTURE.md), [Domain](DOMAIN_MODEL.md), [Integration](OSTRIS_INTEGRATION.md), [Public boundary](PUBLIC_SOFTWARE_BOUNDARY.md) and [Readiness](IMPLEMENTATION_READINESS.md). Run the local instance from sibling `stir-main`.

Version: `0.4.0-SNAPSHOT`. No production deployment yet - a production-like Docker path (object storage, one-shot migrations, reverse-proxy-safe TLS handling, secret-hygiene fail-fast checks) is prepared but not executed against any real destination server. Economic exchange (Agreement -> real osTRIS EXCHANGE -> commit) is implemented for development/validation; see [Integration](OSTRIS_INTEGRATION.md) and [Transaction lifecycle](TRANSACTION_LIFECYCLE.md). 0.4 adds photos, in-app notifications, multi-device credential lifecycle, minimal content moderation and fully configuration-driven instance branding - see [Domain](DOMAIN_MODEL.md)'s "Public pilot additions" section.
