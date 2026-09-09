# Domain model

| Concept | Responsibility / status |
|---|---|
| Marketplace | Instance/tenant commercial scope; future explicit settings, not community identity |
| ParticipantProfile | Future tenant-scoped presence, distinct from IDAX login and osTRIS participant |
| Listing | Publication: tenant, owner, OFFER/WANTED, title, description, category, resource kind, optional coarse location, ACTIVE/CLOSED, version, timestamps |
| ListingType | OFFER or WANTED; independent from resource kind |
| Category | Stable catalog code and localized label |
| Resource | Future reusable item/skill description; no speculative inventory table now |
| Offer | Future proposed response to a listing, not ListingType.OFFER |
| Negotiation | Future access-controlled exchange between parties |
| Agreement | Future accepted commercial terms, not economic completion |
| AgreementSnapshot | Future immutable versioned canonical bytes and private nonce |
| Trade | Future fulfillment linked to osTRIS transaction/receipt |
| Attachment | Future authorized object reference and scan status; no upload now |
| Location | Optional coarse text; precise address reserved for fulfillment |

Listing fields are atomic. No aggregated lists, uploads, negotiations or journal entries in a megaentity. Category and resource-kind codes are relational catalogs. Adding a kind is an additive migration, not a closed economic enum. Initial kinds: physical, service, knowledge, collaboration, volunteering, work, other.

Listings start ACTIVE. Owners may edit ACTIVE listings or close them. CLOSED is terminal; repeated close is idempotent. Version checks reject concurrent edits/close. No physical delete endpoint. Filters include direction, category, kind, state, search text and ownership, with bounded pagination.

Tenant and owner are server-derived and immutable. Tenant-qualified queries and forced PostgreSQL RLS defend each other. Owner remains independent from future participant mapping. No account or balance is stored.

Deferred: multiple marketplaces per tenant, category administration, geography, quantity units, moderation, attachments/retention, participant verification, negotiation and snapshots.
