# STIR

**Sistema Transparente de Intercambio de Recursos** is an open source marketplace built on osTRIS. STIR application source is Apache-2.0.

`stir.es` is a future public reference instance. Anyone can deploy an independent instance with its own hostname, users, database, secrets, tenants and explicitly configured economic communities. No runtime logic depends on that hostname.

STIR uses public IDAX Open Core 0.3 runtime artifacts and the public IDAX Shell. Core is publicly obtainable but separately licensed and is not open source. The entire dependency stack must not be described as Apache-2.0 or reproducible from source: Core's published binary is an explicitly accepted build input.

Start with [Architecture](ARCHITECTURE.md), [Domain](DOMAIN_MODEL.md), [Integration](OSTRIS_INTEGRATION.md), [Public boundary](PUBLIC_SOFTWARE_BOUNDARY.md) and [Readiness](IMPLEMENTATION_READINESS.md). Run the local instance from sibling `stir-main`.

Version: `0.1.0-SNAPSHOT`. No production deployment or economic exchange is included.
