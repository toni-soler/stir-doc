# Transaction lifecycle

Implemented: ACTIVE listing → owner edit or CLOSED. No money movement.

Future: Listing → Offer → Negotiation → Acceptance → Agreement → immutable AgreementSnapshot → Trade pending → osTRIS proposal → authorization/policies → COMMITTED receipt → economic phase completed. Delivery and disputes remain separate.

Policy rejection leaves trade incomplete. Network timeout means unknown, never success. Persist stable intent and reconcile before retry. REVERSAL is another normative transaction, never journal editing. Anchor can remain pending after COMMITTED.
