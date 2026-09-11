# Transaction lifecycle

Implemented (0.1): ACTIVE listing -> owner edit or CLOSED. No money movement.

Implemented (0.2): Listing -> Offer opens a Negotiation -> Counteroffer(s) (append-only, previous offer SUPERSEDED, never edited) -> Accept -> Agreement (unique per Negotiation) + immutable AgreementSnapshot, created together in one transaction. Agreement.economicPhase is AWAITING_ECONOMIC_EXECUTION. No osTRIS call happens as part of accept.

Future: AgreementSnapshot -> osTRIS proposal -> authorization/policies -> COMMITTED receipt -> economic phase completed. Delivery and disputes remain separate. Trade will carry the link between an Agreement and its eventual osTRIS transaction/receipt once the public discovery/status/provisioning gaps in OSTRIS_INTEGRATION.md close.

Policy rejection leaves trade incomplete. Network timeout means unknown, never success. Persist stable intent and reconcile before retry. REVERSAL is another normative transaction, never journal editing. Anchor can remain pending after COMMITTED.
