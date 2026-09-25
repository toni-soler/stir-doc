# Market integrity in STIR

This increment keeps five different facts separate: an osTRIS committed entry, a
STIR Agreement, an immutable raw observation, that observation's eligibility for
a descriptive reference, and the community's published reference. A valid
Agreement can remain in the Agreement history while a later FINAL integrity case
excludes its observation from future reference snapshots. Nothing in the
integrity workflow edits the Agreement or osTRIS ledger.

## Reconstruction path

`reference_observation` is append-only. `reference_policy` and
`market_constitution` are versioned. `market_integrity_case` contains a private
signal and its source observation; immutable `market_integrity_case_event` rows
carry SIGNAL → UNDER_REVIEW → FINAL or DISMISSED. The case originator cannot make
their own FINAL/DISMISSED decision. Only a FINAL event before a daily UTC cutoff
changes eligibility. The snapshot's private evidence manifest records each
included observation ID and excluded ID with a reason such as
`FINAL_INTEGRITY_FINDING:RELATED_PARTICIPANT_CLUSTER`. The source row is never
deleted. The public snapshot contains the policy and constitution versions and
digests, thresholds, method and privacy-safe aggregate when sufficient.

The current method measures observation count, distinct account count, distinct
unordered economic relationships, largest account and pair shares, repeated A↔B
activity, distinct UTC days, freshness and the largest change in the median when
one observation is removed. It does not call an account a person, infer that
separate accounts are independent people, identify circular trading as fraud, or
remove statistical outliers silently. An extreme listing never becomes accepted
Agreement evidence. An extreme real Agreement remains raw evidence; a review
must reach FINAL before exclusion. DISMISSED signals do not exclude anything.

Without enough eligible observations, accounts, relationships or freshness, or
when concentration is high, the API says `INSUFFICIENT_DATA` and suppresses
individual amounts, exact small counts and concentration ratios. Successful
cohorts expose descriptive metrics and a leave-one-out sensitivity, not a fair
price. A normative community reference can still differ deliberately from the
distribution with an explanation. Publication remains a separate decision.

Known identity/continuity clusters can be represented by a reviewed
`RELATED_PARTICIPANT_CLUSTER` case with private evidence references. STIR does not
currently ingest osTRIS RiskSubject IDs or KYC information. An unverified cluster
claim is a signal, not a finding. The private case API uses the existing delegated
publisher permission; ordinary reference readers see only aggregate status and
reason codes. Case references and actor identities are not exposed through the
public reference view. This is not differential privacy.

The constitutional fields require provenance, immutable history, the fixed
7-of-7 threshold and a non-governing Guardian. The operational policy can change
its window, minimum counts, account concentration limit and freshness inside
constitutional floors. The normal policy API rejects requests carrying
independence, concentration, provenance or forced-reference mutations. The
constitutional authority can change the explicit independence/concentration
flags only through a signed 7-of-7 version. Database constraints retain the
minimum privacy floors even if all seven approve a weaker constitution; changing
those schema protections is a separate reviewed software migration.

The detection limits matter: a coalition can manufacture apparently diverse
accounts and relationships; repeated accounts are measurable, real-world
independence is not. Human review and legitimate identity continuity evidence
remain necessary. No fraud score or automatic punitive action is produced.
