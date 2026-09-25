# Market and governance capture: threat model

Market capture changes the observed signal: wash trades, A↔B repetition,
circular activity, concentrated counterparties, and related accounts that look
independent. Governance capture changes which signals count or who can decide:
lowering floors, turning off independence checks, rewriting history, bypassing
publication or capturing recovery. The first is partly statistical; the second
requires explicit authority boundaries and permanent audit.

An attacker can submit genuine Agreements at arbitrary negotiated values. STIR
does not invalidate them or osTRIS transactions. Source tags, bilateral consent,
exact unit comparison, account/pair concentration, temporal spread, sensitivity,
private review events and reason-coded exclusions make their influence
inspectable. An accepted outlier is not automatically discarded. A suspected
cluster remains SIGNAL or UNDER_REVIEW until a separate actor reaches FINAL;
DISMISSED remains eligible. Private cluster evidence is not a public KYC feed.

An ordinary publisher may change reference policy only inside code and
constitution bounds. Protected API fields are rejected. A constitutional
proposal retains its exact before/after diff and partial signatures even if it
never activates. Activation rechecks current seat credentials, current
constitution digest, freeze state and action-specific threshold. An admin HTTP
permission is not a constitutional signature. A Guardian can suspend one key,
but cannot fill that seat's vote; 5/7 can remove a captured Guardian.

Residual threats are explicit. A single human can control many accounts or even
multiple constitutional keys unless an external identity and appointment
process prevents it. STIR does not consume private osTRIS RiskSubject links yet,
so account diversity is not independent-person diversity. Case reviewers can
collude; the event chain makes decisions reconstructible but cannot prove their
motives. A database superuser can defeat application RLS and triggers, so the
database and backups need separate operational controls. Unanimity can make
constitutional changes unavailable, especially while controller-replacement
finality lacks a verified contract. These are not solved by an opaque fraud score
or a superkey.

The engineering observation for the private editorial laboratory is that
**valid exchange**, **eligible evidence** and **legitimate rule change** are
three independent judgments. A system can prove which keys signed a proposal
and which facts were excluded, not that a person was morally corrupt or why a
different person did not sign. This document does not edit or publish the book.
