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
