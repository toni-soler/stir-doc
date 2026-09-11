# Domain model

IDAX tenant != Marketplace. IDAX tenant != osTRIS Community.
Marketplace != osTRIS Community. IDAX user != osTRIS Participant.

0.1/0.2 use one tenant as one marketplace scope only as an implementation simplification, not a domain invariant. A future Marketplace identifier/configuration may distinguish multiple marketplaces within a tenant. Economic associations always require explicit bindings; no equality of these concepts is implied.

| Concept | Responsibility / status |
|---|---|
| Marketplace | Instance/tenant commercial scope; future explicit settings, not community identity |
| ParticipantProfile | 0.2: tenant-scoped presence (display name, short bio, optional coarse location), distinct from IDAX login and osTRIS participant |
| Listing | Publication: tenant, owner, OFFER/WANTED, title, description, category, resource kind, optional coarse location, ACTIVE/CLOSED, version, timestamps |
| ListingType | OFFER or WANTED; independent from resource kind |
| Category | Stable catalog code and localized label |
| Resource | Future reusable item/skill description; no speculative inventory table now |
| Offer | 0.2: a proposal against a Listing (not ListingType.OFFER) - message, optional quantity/unit, optional proposed amount/unit reference, optional terms, status, sequence |
| Negotiation | 0.2: access-controlled Offer thread between a Listing's owner and one initiator; OPEN/ACCEPTED/DECLINED |
| Agreement | 0.2: accepted commercial terms once a Negotiation is accepted - not economic completion; economicPhase starts and currently stays AWAITING_ECONOMIC_EXECUTION |
| AgreementSnapshot | 0.2: immutable, versioned, canonical (sorted-key compact JSON) record of the accepted terms plus a private random nonce, SHA-256 digest over the exact canonical bytes |
| Trade | Future fulfillment linked to osTRIS transaction/receipt |
| Attachment | Future authorized object reference and scan status; no upload now |
| Location | Optional coarse text; precise address reserved for fulfillment |

Listing fields are atomic. No aggregated lists, uploads, negotiations or journal entries in a megaentity. Category and resource-kind codes are relational catalogs. Adding a kind is an additive migration, not a closed economic enum. Initial kinds: physical, service, knowledge, collaboration, volunteering, work, other.

Listings start ACTIVE. Owners may edit ACTIVE listings or close them. CLOSED is terminal; repeated close is idempotent. Version checks reject concurrent edits/close. No physical delete endpoint. Filters include direction, category, kind, state, search text and ownership, with bounded pagination. Read paths (search/read) enrich each row with the owner's ParticipantProfile.displayName; write paths are unchanged.

Tenant and owner are server-derived and immutable. Tenant-qualified queries and forced PostgreSQL RLS defend each other. Owner remains independent from future participant mapping. No account or balance is stored.

## Negotiation lifecycle (0.2)

An Offer against an ACTIVE Listing (by anyone but its owner) opens a Negotiation and is its first, sequence-1 Offer. A party may only have one OPEN Negotiation per Listing at a time. Only the two parties (the Listing's owner and the Negotiation's initiator) may read or act on it; everyone else gets 404, never 403, to avoid existence leaks.

Offers are append-only and never edited: a counter-offer creates a new Offer row, marks the previous head SUPERSEDED, and becomes the new head. The author must alternate - a party cannot counter or accept their own last offer. Every mutating action (counter/accept/decline) carries the caller's last-seen Negotiation.version; a mismatch, a Negotiation that is no longer OPEN, an offerId that is no longer the current head, or the underlying Listing no longer being ACTIVE (accept/counter only) all fail with 409, never silently.

Accepting freezes the head Offer's status to ACCEPTED, closes the Negotiation (ACCEPTED), and creates exactly one Agreement (unique per Negotiation) plus its AgreementSnapshot in the same transaction. The Agreement's economicPhase is AWAITING_ECONOMIC_EXECUTION: an Agreement is a commercial commitment between STIR parties, never itself an osTRIS economic completion (see TRANSACTION_LIFECYCLE.md/OSTRIS_INTEGRATION.md).

Deferred: multiple marketplaces per tenant, category administration, geography, quantity units, moderation, attachments/retention, participant verification, Trade/osTRIS EXCHANGE integration.
