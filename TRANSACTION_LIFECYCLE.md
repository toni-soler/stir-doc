# Transaction lifecycle

Implemented (0.1): ACTIVE listing -> owner edit or CLOSED. No money movement.

Implemented (0.2): Listing -> Offer opens a Negotiation -> Counteroffer(s) (append-only, previous offer SUPERSEDED, never edited) -> Accept -> Agreement (unique per Negotiation) + immutable AgreementSnapshot, created together in one transaction. Agreement.economicPhase is AWAITING_ECONOMIC_EXECUTION. No osTRIS call happens as part of accept.

Implemented (0.3): if the accepted Offer has a proposedAmount, Agreement.economicPhase becomes AWAITING_ECONOMIC_EXECUTION with payerUserId/payeeUserId frozen from Listing.direction (OFFER: initiator pays owner; WANTED: owner pays initiator); otherwise it becomes NOT_APPLICABLE (free/non-monetary, nothing further to do). From AWAITING_ECONOMIC_EXECUTION: activate (creates the Trade + osTRIS proposal, -> AWAITING_SIGNATURES) -> both parties sign with their own client-side Ed25519 key -> commit (calls osTRIS; COMMITTED with the receipt's committedSequence/protocolDigest/committedAt recorded, or REJECTED on a directly-observed osTRIS failure). `sync()` independently re-reads osTRIS's own status for reconciliation, never trusting a client claim. Delivery and disputes remain separate. See DOMAIN_MODEL.md's "Economic exchange lifecycle" and OSTRIS_INTEGRATION.md for the full contract.

Policy rejection leaves trade incomplete - the Agreement still exists, the Trade is REJECTED, no balance moves, and STIR never overrides osTRIS's decision. Network timeout means unknown, never success; STIR persists a stable transactionId (its own UUIDv7) and reconciles through sync() before assuming failure. REVERSAL is another normative transaction, never journal editing. Anchor can remain pending after COMMITTED.
