# Architecture

```text
User -> STIR Frontend (public IDAX Shell extension)
     -> STIR Backend (commercial data)
     -> osTRIS API (future economic lifecycle)
     -> IDAX Ledger (evidence)
     -> XRPL private network (optional future provider)
```

Frontend shares Shell authentication, tenant selection, navigation, locale and authenticated HTTP. Backend consumes the published IDAX Core binary for authentication, tenant context, permissions and transaction-scoped RLS. PostgreSQL owns the stir schema and Flyway history. osTRIS and Ledger remain independent processes; STIR has no dependency on their Java implementation or database tables.

The development composition includes Shell, STIR, osTRIS and Ledger with public source builds. Proof delivery and XRPL remain disabled pending service-principal/provider provisioning. Listing CRUD requires neither community nor economic operation.

Decisions: Java 21, Boot 3.4.4, PostgreSQL 17, React 18 JavaScript, esbuild extension. Public 0.3 uses JavaScript, not TypeScript. No private module scaffolder is used: public reproducibility and the request to avoid unnecessary generated code take precedence over internal DevKit conventions.

Ownership is the authenticated IDAX user in the selected tenant. Administrators cannot edit another owner's listing through owner endpoints. Tenant never implies an osTRIS community. Future MarketplaceCommunityBinding and ParticipantBinding require explicit administration and verified identifiers.

0.2 adds four more `stir` schema tables behind the same defense-in-depth (tenant-scoped application queries + forced PostgreSQL RLS + explicit server-side party/ownership checks): `participant_profile`, `negotiation`, `offer` and `agreement`/`agreement_snapshot`. Negotiation party authorization (only the Listing's owner and the Negotiation's initiator) is enforced in `NegotiationService`/`AgreementService`, not in RLS - RLS in this project only ever enforces tenant isolation, matching the Listing precedent audited in AUTHENTICATION_AUDIT.md. See DOMAIN_MODEL.md for the negotiation state machine and OSTRIS_INTEGRATION.md for where AgreementSnapshot fits against the future osTRIS EXCHANGE call.

Shell 0.3 hardcodes two module hosts and does not export its CRUD component. A pinned-source adapter in stir-main generalizes mounting and exposes active tenant context. The platform correction belongs to an independent idax-shell branch; stir-main applies its reviewed public-source patch temporarily.

0.3 adds real economic exchange. STIR's backend is the SOLE caller of osTRIS's HTTP API (`OstrisClient`, relaying the end user's own bearer token - never a separate service credential, never direct SQL or a shared JPA repository): `User -> STIR Frontend -> STIR Backend -> osTRIS public API -> IDAX Ledger`. Two new `stir` schema tables, `marketplace_economic_binding` (tenant -> osTRIS community/unit, one per tenant) and `participant_economic_binding` (user -> osTRIS participant/account/credential/controller), make that association explicit and auditable - never inferred from UUID equality. `Trade` is STIR's own link to one osTRIS EXCHANGE transaction per Agreement; its `executionState` reflects only what osTRIS itself returned from a direct commit call or a `sync()` re-read of osTRIS's own status, never an unverified client claim. Ed25519 signing keys are generated client-side (WebCrypto, non-extractable where the platform allows) and persisted per-user in that browser's own IndexedDB; STIR and osTRIS never see or generate a private key, and the client signs only the exact canonical bytes osTRIS itself hands back (`GET .../trade/signing-payload`), never a client-reconstructed payload.
