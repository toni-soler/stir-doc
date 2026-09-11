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
| Agreement | 0.2: accepted commercial terms once a Negotiation is accepted - not economic completion. 0.3: economicPhase is NOT_APPLICABLE (no proposed amount - free/non-monetary), or AWAITING_ECONOMIC_EXECUTION -> AWAITING_SIGNATURES -> COMMITTED/REJECTED once a Trade is activated. payerUserId/payeeUserId are derived once, at accept time, from Listing.direction (OFFER: initiator pays owner; WANTED: owner pays initiator) - never re-inferred later |
| AgreementSnapshot | 0.2: immutable, versioned, canonical record of the accepted terms plus a private random nonce, SHA-256 digest over the exact canonical bytes. 0.3: canonicalization is RFC 8785 JCS (`STIR-AGREEMENT-JCS-1`, via `io.github.erdtman:java-json-canonicalization` / `canonicalize` on JS - the same library osTRIS's own reference implementation uses), schemaVersion 2, adding `format` and nullable `payerUserId`/`payeeUserId` fields |
| MarketplaceEconomicBinding | 0.3: explicit, one-per-tenant tenant -> osTRIS community/unit binding; set once via an admin-style bootstrap action, never inferred |
| ParticipantEconomicBinding | 0.3: explicit user -> osTRIS participant/account/credential/controller binding, created by activating economic exchange with a client-generated Ed25519 public key |
| Trade | 0.3: STIR's own link between one Agreement and one osTRIS EXCHANGE transaction - transactionId, payer/payee account, amount (minor units), contractualMetadataDigest (== the AgreementSnapshot digest), executionState (AWAITING_SIGNATURES/COMMITTED/REJECTED), committedSequence/protocolDigest/committedAt once committed |
| Attachment | Future authorized object reference and scan status; no upload now |
| Location | Optional coarse text; precise address reserved for fulfillment |

Listing fields are atomic. No aggregated lists, uploads, negotiations or journal entries in a megaentity. Category and resource-kind codes are relational catalogs. Adding a kind is an additive migration, not a closed economic enum. Initial kinds: physical, service, knowledge, collaboration, volunteering, work, other.

Listings start ACTIVE. Owners may edit ACTIVE listings or close them. CLOSED is terminal; repeated close is idempotent. Version checks reject concurrent edits/close. No physical delete endpoint. Filters include direction, category, kind, state, search text and ownership, with bounded pagination. Read paths (search/read) enrich each row with the owner's ParticipantProfile.displayName; write paths are unchanged.

Tenant and owner are server-derived and immutable. Tenant-qualified queries and forced PostgreSQL RLS defend each other. Owner remains independent from future participant mapping. No account or balance is stored.

## Negotiation lifecycle (0.2)

An Offer against an ACTIVE Listing (by anyone but its owner) opens a Negotiation and is its first, sequence-1 Offer. A party may only have one OPEN Negotiation per Listing at a time. Only the two parties (the Listing's owner and the Negotiation's initiator) may read or act on it; everyone else gets 404, never 403, to avoid existence leaks.

Offers are append-only and never edited: a counter-offer creates a new Offer row, marks the previous head SUPERSEDED, and becomes the new head. The author must alternate - a party cannot counter or accept their own last offer. Every mutating action (counter/accept/decline) carries the caller's last-seen Negotiation.version; a mismatch, a Negotiation that is no longer OPEN, an offerId that is no longer the current head, or the underlying Listing no longer being ACTIVE (accept/counter only) all fail with 409, never silently.

Accepting freezes the head Offer's status to ACCEPTED, closes the Negotiation (ACCEPTED), and creates exactly one Agreement (unique per Negotiation) plus its AgreementSnapshot in the same transaction. The Agreement's economicPhase is AWAITING_ECONOMIC_EXECUTION: an Agreement is a commercial commitment between STIR parties, never itself an osTRIS economic completion (see TRANSACTION_LIFECYCLE.md/OSTRIS_INTEGRATION.md).

Deferred: multiple marketplaces per tenant, category administration, geography, quantity units, moderation, attachments/retention, participant verification.

## Economic exchange lifecycle (0.3)

Activating a Trade (`POST .../trade/activate`, either party, once) derives entries from STIR's own Agreement/Offer data (never a client-supplied amount), creates the osTRIS EXCHANGE proposal server-to-server, and moves Agreement.economicPhase to AWAITING_SIGNATURES. Each party fetches the exact canonical bytes to sign from osTRIS (`GET .../trade/signing-payload`) and submits their own client-produced Ed25519 signature (`POST .../trade/authorizations`) - STIR only relays it, never signs on a user's behalf. Commit (`POST .../trade/commit`) calls osTRIS directly and observes the result: COMMITTED on success (committedSequence/protocolDigest/committedAt recorded, Agreement -> COMMITTED), or REJECTED on a directly-observed osTRIS failure (e.g. a credit-floor breach) - recorded in its own transaction so it survives even though the failing request itself rolls back, and NEVER fabricated from a client's unverified claim. `sync()` independently re-reads osTRIS's own transaction status for reconciliation. Activate/commit are idempotent (repeating them never creates a second Trade or a second osTRIS journal entry). Balances shown to a participant are always live-queried from osTRIS, never cached as STIR's own source of truth.
