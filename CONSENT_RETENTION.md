# Consent and retention for reference evidence

## Four questions, never collapsed into one

STIR already recorded every AGREEMENT-sourced offer into `stir.reference_observation`
regardless of consent (`aggregate_consent` gated *use*, never *storage* -
`COMMUNITY_VALUE_REFERENCES.md`). This increment names and separates the four distinct
questions that were previously implicit:

1. **Having a datum** - the observation row exists (`reference_observation`), always,
   for every AGREEMENT-sourced offer, whether or not anyone consented to anything.
2. **Having permission to use it** - `stir.reference_consent`/`reference_consent_event`:
   purpose-specific, per-party, grant/decline/withdraw, with a full audit trail.
3. **Still being allowed to retain it** - `stir.retention_policy`: a versioned,
   community-scoped window after which the *identifiers* (not the datum) may be
   removed, unless a real hold applies.
4. **Still being eligible as current evidence** - `EvidenceAnalysis`'s existing live
   exclusion mechanism (`NO_BILATERAL_CONSENT`, now also `CONSENT_WITHDRAWN`), computed
   fresh for every not-yet-cached daily cutoff, never rewriting a cached one.

`NOT_ELIGIBLE_FOR_NEW_USE` (question 2/4), `MUST_BE_RETAINED` (a hold - question 3), and
`ELIGIBLE_FOR_ANONYMIZATION`/`ANONYMIZED` (the terminal state of question 3) are three
separately computed, never-conflated states (`RetentionService.RetentionStatus`). An
observation can be simultaneously `CONSENT_WITHDRAWN` (ineligible for new use) and
`MUST_BE_RETAINED` (an open market-integrity case still needs its identifiers) - consent
withdrawal never deletes anything, and retention never overrides a live investigation.

## Consent: purpose-specific, captured once, withdrawable once

`purpose` is a real column with exactly one value today, `REFERENCE_EVIDENCE_CONTRIBUTION`
- deliberately not a generic "I accept everything" consent, and deliberately not
pretending other purposes exist yet. Grant/decline is captured **automatically**, at
Agreement acceptance, from the pre-existing bilateral `shareReferenceObservation` flow
(`ReferenceAcceptanceAdapter.accepted()`): the offeror's decision is `Offer.shareReferenceObservation`
(set when they made this exact offer), the acceptor's is the flag they pass to
`accept()`. Each party's own decision is now recorded as its own row
(`stir.reference_consent`, one per `(observation, party, purpose)`), not only folded into
the single `aggregate_consent` boolean the observation itself carries - so either party
can later see and withdraw *their own* decision without needing to know the other's.

**Withdrawal is the only human action here**, and it is a personal, self-service right:
`ConsentService.withdraw()` is gated only on being the consenting party (`party_user_id`),
never on `stir.references.publish` or ordinary-governance authority. It is idempotent
(an advisory lock plus a status check - a repeated or replayed withdrawal returns the
same state, never a second event) and it can never be reversed back to `GRANT`. Declining
in the first place is not withdrawable (`"Consent was declined, not granted"`) since
there is nothing to withdraw.

**What withdrawal actually changes**: `ReferenceService.snapshot()` queries the latest
`reference_consent_event` per observation and folds any `WITHDRAW` into the same
`finalExclusions` map that already carried Market Integrity's `FINAL_INTEGRITY_FINDING`
reason (`ReferenceService.java`), with a new, distinct reason - `CONSENT_WITHDRAWN` -
`putIfAbsent`'d so a FINAL finding always wins the reason shown. This is the entire
mechanism: no change to `EvidenceAnalysis.java`'s pure function at all. `aggregate_consent`
itself, frozen on the observation at acceptance time, is **never rewritten** - the same
freeze-and-never-mutate convention as everything else in this bounded context.

Because a `reference_snapshot` already cached for a given UTC cutoff is returned as-is
(`ReferenceService.snapshot()`'s existing daily cache, unchanged), a withdrawal can only
ever affect a **future**, not-yet-computed cutoff. A reference already published, and
the evidence snapshot it cited, stay exactly as they were - reproducible with the policy
and evidence valid at the time, forever (`cachedSnapshotSurvivesALaterWithdrawalUnchanged`).
A *new* reference proposal, computed after the withdrawal, correctly excludes it.

## Retention: a versioned policy, one real deletion action

`stir.retention_policy` is versioned and immutable, community-scoped (like
`ordinary_governance_policy`/`community_governance_member`, not per-definition), with
exactly the fields STIR's own lifecycle needs: `retention_period_days`, `basis` (one
value today, `COMMUNITY_REFERENCE_AND_INTEGRITY_HISTORY`), `explanation`. **Not** a
legal/compliance engine - no jurisdiction, no data-category taxonomy, no configurable
deletion-vs-anonymization choice. The one real action is anonymization.

**The floor is a code constant, not a Seven Keys field.** `RetentionService.MINIMUM_RETENTION_PERIOD_DAYS = 90`
is enforced the same way `REPLACE_CONTROLLER` is fail-closed - by construction, not by a
constitutional amendment path. This was a deliberate choice against extending Seven
Keys' constitution schema (`SevenKeysService.initialConstitution()`/`validateConstitution()`):
that schema requires an exact key-set match everywhere a 7-of-7 amendment is proposed,
so adding a field there has a real blast radius (every existing amendment proposal,
every test, the signer UI) for a floor that does not need cryptographic 7-of-7
protection to be real - a hardcoded minimum a service method simply refuses to go below
achieves the same "not an ordinary parameter" guarantee with none of that risk. If a
future increment genuinely needs a *community-adjustable* retention floor gated by
Seven Keys rather than a fixed code constant, that is an open **SPEC GAP**, not
something this increment silently assumed.

**Anonymization removes only `participant_a`/`participant_b`.** `reference_observation`
was, before this increment, part of the same fully append-only table set as every other
table in this bounded context (`reject_reference_mutation()`, `GRANT SELECT,INSERT` only
- V5). Rather than weaken that for the whole table, this increment gives it its own
dedicated trigger function, `reject_reference_observation_mutation()`, that permits
**exactly one transition**: `participant_a`/`participant_b` moving from their original
value to `NULL` and `anonymized_at`/`anonymized_by` moving from `NULL` to a real value,
with every other column - `amount`, `quantity`, `observed_at`, `aggregate_consent`,
`source`, `id` - required to stay byte-identical, and any `DELETE` always rejected. A raw
SQL attempt to touch anything else, or to anonymize a second time, fails at the trigger,
not just at the service layer (`rawSqlCannotBypassTheAnonymizationOnlyTrigger`). The
audit trail is a genuinely separate, fully immutable table (`retention_lifecycle_event`),
not just the mutated row's own two new columns.

**Eligibility for anonymization is `RetentionService.statusFor()`, computed live** -
same "compute on read, never persist as its own status" convention as
`OrdinaryGovernanceService.lazyClose()`/`isStale()`:

```
if anonymized_at is set:                      ANONYMIZED
elif a market_integrity_case exists for it:    MUST_BE_RETAINED (hold=true)
elif now < observed_at + retention_period_days: RETAINED
else:                                           ELIGIBLE_FOR_ANONYMIZATION
```

The hold check is deliberately **any** case, in any status (`SIGNAL`/`UNDER_REVIEW`/
`FINAL`/`DISMISSED`) - an investigation still needs the identifiers even before it
reaches `FINAL`. There is no manual "governance hold" flag in this increment: the only
real, evidenced basis for a hold today is an existing market-integrity case, so that is
the only basis this increment implements - a generic hold-setting API with no real
reason to hold anything yet would be exactly the "mega retention policy" the brief
explicitly warned against. `anonymize()` never deletes and never overrides that hold;
attempting it while held is a 409 naming the hold reason, never a silent no-op.

`dueForAnonymization(definitionId)` is a **review list, not a trigger** - listing
eligible observations never anonymizes them. Nothing in this codebase runs a scheduled
job; anonymization is an explicit, permission-gated action a publisher takes on one
observation at a time, exactly the "no borrar silenciosamente" the brief required.

## Market Integrity: withdrawal can never erase evidence of manipulation

Two independent guarantees, neither new code beyond what's described above:

- `MarketIntegrityService.signal()`/case creation query `reference_observation` by id
  directly, with no consent filter at all - consent status never gated whether an
  observation could be investigated, before or after this increment.
- The exclusion-reason priority in `ReferenceService.snapshot()` (`finalExclusions.putIfAbsent`)
  means a `FINAL_INTEGRITY_FINDING` reason is never overwritten by a later
  `CONSENT_WITHDRAWN` - withdrawing consent on an already-FINAL-flagged observation
  changes nothing about how it is reported (`finalIntegrityFindingTakesPriorityOverWithdrawalReason`).
- Retention's hold (above) means "erase the evidence" is not just *reported* correctly
  but structurally *impossible* while a case is open: `anonymize()` refuses.

"Retiro consentimiento" can never mean "elimino las pruebas de mi comportamiento
anterior" - it was already true for the raw Agreement/observation (never touched by
consent withdrawal), and this increment makes it equally true for the one action that
*could* touch the observation, anonymization.

## Participant Independence: no new sensitive data kept

Anonymization removes exactly the identifiers `EvidenceAnalysis`'s diversity/independence
computation would otherwise use - once anonymized, the observation falls out of
`participants`/`pairs` entirely on the very next live computation (`o.a()==null` triggers
the pre-existing `MISSING_COUNTERPARTY` exclusion; no change to `EvidenceAnalysis.java`).
An anonymized account never becomes falsely "known independent" or falsely "known
related" - it simply stops contributing to that observation's coverage at all, the same
`UNKNOWN != INDEPENDENT` discipline `PARTICIPANT_INDEPENDENCE.md` already established,
now also holding across a retention/anonymization boundary, not only a refresh-coverage
one. No new participant identifier is retained anywhere beyond what
`reference_observation`/`reference_consent` already needed.

## Ordinary Governance: retention policy is governable, within the floor

`RETENTION_POLICY_CHANGE` is a new `ordinary_proposal.proposal_type` (this required
widening `ordinary_proposal.definition_id` to nullable - retention policy is
community-scoped, like the electorate/voting-policy tables, not definition-scoped like
the two existing proposal types; the `proposal_type` CHECK constraint was extended, not
replaced). `OrdinaryGovernanceService.proposeRetentionPolicyChange()`/`execute()` freeze
and dispatch it exactly like `REFERENCE_POLICY_CHANGE`, reusing `RetentionService.setPolicyDirect()`
(the floor check applies identically whether the call came from a direct publisher
action or an approved vote) with its own staleness digest
(`retentionPolicyStateDigest`, parallel to but never confused with `policyStateDigest`).
Once a community enables ordinary governance, `RetentionService.setPolicy()`'s existing
gate (`requireDirectMutationAllowed`, the same one `ReferenceService.policy()` already
used) blocks a direct change - without this proposal type, retention policy would have
**silently frozen forever** the moment governance was enabled, which is exactly the
sort of half-wired gap this increment closes rather than leaves for later.

What is **not** made an ordinary parameter, because it is constitutional-adjacent by the
brief's own rule ("cambios que reduzcan invariantes... no deben convertirse en simples
parámetros ordinarios"): the 90-day floor itself. A community vote can raise its own
retention period arbitrarily high, never below the floor - `setPolicyDirect()` enforces
this identically regardless of caller.

## Extensions: no bypass, no unlifecycled PII

Per `COMMUNITY_EXTENSION_GUIDE.md`'s existing boundary (`distribution → STIR`, never the
reverse; no SQL/privileged credentials to an extension), this increment adds nothing
that weakens it, and closes one real gap it left open: `externalContractNamespace`/
`externalContractDigest` (`MIXED_CONSIDERATION_EXTENSION.md`) are **opaque** on the STIR
side by design - STIR never stores or interprets what a distribution puts in its own
namespace/digest, and this increment does not change that contract. A distribution that
embeds PII behind that opaque digest is doing so in **its own** storage, under **its
own** consent/retention lifecycle - STIR's consent/retention tables never reference or
gate on those fields, and a distribution gets no automatic right to reuse STIR's own
consent/retention primitives for its own data. If a future distribution genuinely needs
a shared consent/retention primitive across STIR and its own domain, that is an explicit
**SPEC GAP**, not something this increment quietly assumed by omission.

## Deliberately not built

- **A generic legal/compliance engine.** No jurisdiction model, no data-category
  taxonomy, no configurable retention-vs-deletion choice per field.
- **A manual "governance hold" API.** The only real, evidenced hold basis today is an
  existing market-integrity case; a generic hold-setting mechanism with no other real
  reason to hold anything would be speculative, not needed.
- **Scheduled/automatic anonymization.** Always an explicit, one-observation-at-a-time,
  permission-gated action; `dueForAnonymization()` only ever lists candidates.
- **Attachment/photo retention.** This increment scopes retention to
  `reference_observation`; photo/attachment lifecycle is a separate, not-yet-addressed
  surface.
- **A Seven-Keys-amendable retention floor.** The floor is a code constant
  (see "Retention" above) - a real SPEC GAP if a future increment needs otherwise.
- **A shared consent/retention primitive for extensions.** See "Extensions" above.

## Validation

Backend: `mvn verify` - `ConsentRetentionPostgresTest` (16 tests): bilateral consent
contributes to evidence; a declined party's observation is recorded but never eligible,
and cannot be "withdrawn" since it was never granted; withdrawal excludes with its own
reason and is exclusive to the consenting party; withdrawal is idempotent under
concurrent/replayed calls (exactly one event, never two); a cached snapshot survives a
later withdrawal unchanged; a FINAL integrity finding always outranks a withdrawal
reason; retention RETAINED vs ELIGIBLE_FOR_ANONYMIZATION vs MUST_BE_RETAINED (market
integrity hold) are distinct and correctly computed; anonymization removes only
identifiers, preserves amount/history, records an audit event, and cannot repeat; raw
SQL cannot bypass the anonymization-only trigger for any other column or a second
anonymization; the retention floor is enforced; platform SuperAdmin is excluded from
both retention policy changes and anonymization; tenant isolation holds for consent and
retention; ordinary governance can change retention policy through an approved vote,
and a direct change is correctly blocked once governance is active for that community.

HTTP E2E (`stir-main/scripts/consent_retention_e2e.py`): the same scenarios against a
real running backend and real negotiation/Agreement acceptance flow, including a
market-integrity signal blocking anonymization even past the retention window, and
cross-tenant isolation on consent records.

Browser E2E (`stir-main/scripts/consent_retention_browser.py`): a real party viewing
their own consent list and withdrawing one with the plain-language impact notice shown
first, and a publisher viewing/using the retention panel - all through the rendered UI.
