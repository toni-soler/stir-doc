# STIR ↔ osTRIS integration

## IMPLEMENTED NOW (public osTRIS 0.3)

Verified against [public osTRIS](https://github.com/toni-soler/ostris) commit d92aa1f605884ebdcd8e97fff46f4067b0416bcc: TransactionController, ProposalAuthorizationService, CommitReceipt and CORE_WIRE_AND_DECISION_SEMANTICS_V0_1.md.

| Public operation | Contract |
|---|---|
| POST /api/ostris/transactions/proposals | communityId, unitId, transactionId, purpose, entries[{accountId,amount}], references, optional contractualMetadataDigest, resolutionBasisType, resolutionBasisId |
| POST /api/ostris/transactions/{id}/authorizations | accountId, credentialId, signatureBase64url |
| POST /api/ostris/transactions/{id}/governance-authorizations | authorityId, authorityPolicyVersion, resolutionBasisType, resolutionBasisId, coveredAccounts, credentialId, signatureBase64url |
| POST /api/ostris/transactions/{id}/commit | Receipt: transactionId, communitySequence, protocolDigest, committedAt |

Permissions: OSTRIS_TRANSACTION_CREATE, OSTRIS_TRANSACTION_AUTHORIZE, OSTRIS_TRANSACTION_COMMIT. Proposal response: transactionId, authorizationDigest, status. Economic IDs obey UUIDv7; amounts are signed integer minor-unit strings. Application 0.3.0 retains normative OSTRIS-CORE-JCS-1 / ostrisCoreVersion 0.1.

Account authorization is Ed25519 evidence checked against effective AccountControlPolicy and credential/controller bindings. Shell login is not this signature. Governance cannot substitute ordinary EXCHANGE authorization. Commit evaluates policies and writes the journal. Reusing a transaction ID with different intent is rejected.

contractualMetadataDigest is optional lowercase SHA-256 hex in participant authorization. references permits only purpose-defined keys: do not invent listingId/tradeId keys. Ordinary EXCHANGE starts with {}; STIR retains its transaction mapping.

## PROPOSED / FUTURE (not implemented in STIR 0.1)

STIR SHALL NOT modify osTRIS balances or economic state directly. Every economic state transition MUST occur through the normative osTRIS transaction lifecycle.

STIR commercial data SHALL NOT become osTRIS journal data except for normative references and cryptographic commitments explicitly defined by the STIR↔osTRIS integration contract.

Explicit tenant/marketplace → community and tenant/user → participant bindings require verification. UUID equality or matching labels do not establish identity. Obtain participants, units, accounts and capability/state through public APIs once supplied. No discovery/read controllers were found in public 0.3, nor GET proposal/status or webhook. No SQL workaround or guessed route is acceptable.

Before proposing EXCHANGE, freeze AgreementSnapshot. Proposed encoding: JCS UTF-8 of a closed versioned object with opaque agreement ID, terms, unit/account references and cryptographically random private nonce. SHA-256 hashes exact canonical bytes. Preserve bytes, nonce, version and digest. Snapshot schema and test vectors require review; osTRIS does not define the external document schema. Never hash mutable Listing JSON as a contract.

**Implemented in 0.2** (`AgreementSnapshotService`/`CanonicalJson`, stir-backend): accepting a Negotiation freezes `{schemaVersion, agreementId, negotiationId, listingId, listingDirection, initiatorId, ownerId, acceptedOfferId, message, quantity, unitLabel, proposedAmount, proposedUnitRef, terms, acceptedAt, nonce}` as a sorted-key, compact-separator, UTF-8 JSON object (STIR's own fixed-schema canonicalizer, not a general JCS/RFC 8785 library - the schema is closed and code-controlled, so key order and number formatting are already deterministic by construction; amounts/quantities are serialized as decimal strings to avoid floating-point ambiguity). `digestSha256 = SHA-256(canonical bytes)`. `CanonicalJsonTest` carries the test vectors: identical fields (including a fixed nonce) reproduce identical bytes/digest regardless of Map insertion order; changing any one contractual term, or the nonce alone, changes the digest. The two Agreement parties can read `canonicalJson`/`digestSha256`/`schemaVersion` (so each can independently recompute the digest); nobody else can reach the Agreement at all (404). `nonce` is never returned as its own top-level API field - see `AgreementSnapshotView`.

Future STIR retains snapshot ID/digest, community, transaction ID, request intent and commit receipt. Timeout means unknown/pending; reuse stable intent and reconcile through a future status contract. Acceptance is not COMMITTED - 0.2's `Agreement.economicPhase` stays `AWAITING_ECONOMIC_EXECUTION` and nothing calls osTRIS yet. No fake gateway or duplicated economic DTOs are provided.

Ledger already anchors committed osTRIS proofs through its outbox/HTTP adapter. STIR must not reproduce proof generation or anchor logic. Anchor and COMMITTED are separate.

Gaps belong to osTRIS: discovery/read APIs, canonical signing payload retrieval, status/reconciliation and public provisioning. Shell SDK gaps belong to idax-shell. Economic integration remains unavailable until these contracts close.
