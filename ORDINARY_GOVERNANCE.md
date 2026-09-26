# Ordinary community governance

## What this replaces, and what it does not

STIR's delegated-publisher model (`stir.references.publish`) is not removed. This
increment adds a real, quorum-based, non-constitutional decision process as an
**explicit opt-in per community** (`stir.community_governance_settings.ordinary_governance_enabled`,
default `false`). Until a community turns it on, `ReferenceService.publish()`/`.policy()`
behave exactly as before - every existing test, script and delegated-publisher workflow
in this codebase is unaffected. Once a community opts in, publishing a reference or
changing its policy for that community requires an **approved ordinary vote**, and a
direct publisher call is refused (409).

This is explicitly **not** Seven Keys. Seven Keys protects the constitutional floors
(the numeric bounds ordinary policy can never cross, and the protected fields -
`independenceChecksRequired`, `concentrationChecksRequired`, `provenanceRequired`,
`forceReference` - it alone can change) via a 7-of-7 cryptographic ceremony. Ordinary
governance is the community's own day-to-day decision process *inside* those floors,
via majority vote, never a constitutional amendment. Both concepts have always
existed in this codebase (`GOVERNANCE_CAPTURE_THREAT_MODEL.md`); this increment gives
the "ordinary" half of that distinction a real mechanism instead of only a delegated
publisher.

## Who votes, and why platform capability grants nothing here

`stir.community_governance_member` is a **wholly STIR-owned electorate roster** -
explicitly not derived from IDAX Core/Shell roles or permissions. Nothing in idax-core
exposes a live "everyone holding permission P" query STIR could safely call without a
new elevated cross-service credential (verified before designing this: idax-shell's own
`GET /roles`, `/roles/{id}/permissions`, `/roles/users` admin endpoints are gated on
`system.roles.read`, an IDAX Core/Shell admin permission an ordinary community member
does not hold, and no STIR code has ever called them). Building the electorate as
STIR's own explicit, add/remove-managed roster avoids that entirely - no new
cross-service trust boundary, same principle already applied to Participant
Independence's refresh design.

Because membership is explicit and STIR-owned:

- **Platform SuperAdmin does not vote by being SuperAdmin.** `ReferenceService.requireCommunityAuthority`
  (already refuses `isSuperuser()`) gates every mutation here exactly like every other
  sensitive mutation in this bounded context.
- **The Guardian does not vote by being Guardian, and a constitutional seat does not
  vote by holding a seat.** Seven Keys' tables (`constitutional_seat`,
  `constitutional_authority`) are never consulted by this service at all - the only
  way into the electorate is an explicit `addMember` call.
- Any of those roles *can* vote, but only if a manager separately, explicitly adds
  them to `community_governance_member` - the same as anyone else.

## Voting policy v0.1 - one explicit shape, not a DSL

`stir.ordinary_governance_policy`, versioned and immutable (append-only, same
convention as `reference_proposal`/`community_reference`):

| Field | Meaning |
|---|---|
| `quorum_numerator`/`quorum_denominator` | fraction of the electorate that must participate |
| `approval_numerator`/`approval_denominator` | fraction of participating (non-abstaining) votes that must approve |
| `voting_window_hours` | 1-2160 (90 days max); how long a proposal stays open |
| `abstention_rule` | `COUNTS_TOWARD_QUORUM_NOT_APPROVAL` or `DOES_NOT_COUNT_TOWARD_QUORUM` |

**Quorum and approval are explicitly distinguished** (`ELIGIBLE VOTERS` ≠ `QUORUM` ≠
`APPROVAL THRESHOLD`, exactly as required): the electorate is who *may* vote; quorum is
how many *must* participate before a result means anything; approval threshold is what
fraction of *those who took a position* must agree. The only numeric floor enforced at
the DB level is that approval must mean a real majority
(`approval_numerator * 2 > approval_denominator`, i.e. strictly over 50%) - "quorum
alone" can never approve anything. Every other number (quorum fraction, window length)
is the community's own explicit, auditable choice, set via `POST .../policy/{communityId}`
by a `stir.governance.manage` holder - not a constitutional floor, and not a hardcoded
default this increment silently imposes. There is no seed/initial value inserted
automatically; a community must explicitly set its first policy before it can create a
governed proposal, exactly the "configuración inicial explícita y auditable" the brief
asked for.

## Proposal, freeze, vote, close, execute

Two proposal types, both frozen exactly once at creation, never recomputed:

- **`PUBLISH_REFERENCE`**: wraps an *already-created*, already-immutable `reference_proposal`
  (created the normal way, via `ReferenceService.propose()`, unchanged). Because
  `reference_proposal` rows are immutable by their own existing trigger, "if content
  changes, a new proposal is required" holds for free - a changed field is necessarily
  a different row, hence a different vote. The ordinary proposal's `payload_digest` is
  a digest over `{referenceProposalId, kind, snapshotId}`.
- **`REFERENCE_POLICY_CHANGE`**: has no pre-existing frozen draft (a direct policy
  change applies immediately), so this increment freezes the exact operational
  `PolicyRequest` fields (`windowDays`, `minimumObservations`, `minimumParticipants`,
  `maximumParticipantShare`, `freshnessDays`, `explanation`) as `payload_json` at
  proposal-creation time.

Both types also freeze `reference_state_digest` (the definition's current policy
version/id) and `electorate_digest` (the frozen roster's own digest) at creation.

**Electorate snapshot**: `stir.ordinary_proposal_electorate` copies the *current*
active `community_governance_member` roster into an immutable, insert-once table the
instant a proposal is created. A member removed after that still votes (they were
eligible when voting opened); a member added afterward cannot (they were not on the
frozen roll) - proven directly (`userRemovedAfterVotingOpensCanStillVoteFromTheFrozenElectorate`,
`userAddedAfterVotingOpensCannotVoteOnAnAlreadyFrozenElectorate`).

**Votes** (`stir.ordinary_vote`) are immutable, one per `(proposal, voter)` - a unique
constraint plus an explicit pre-check make a duplicate vote a 409, not a silent
overwrite. There is no vote-changing endpoint.

**Close is lazy and deterministic, never an administrative override.** There is no
`approveProposal(id)` anywhere in this codebase - the only path to `APPROVED`/
`REJECTED`/`EXPIRED` is `lazyClose()`'s tally, computed purely from the frozen
electorate size and the recorded votes, run whenever anyone reads or votes on a
proposal past its deadline:

```
participants = approve + reject + (abstain if abstention_rule counts it toward quorum)
quorumMet    = participants * quorumDenominator >= quorumNumerator * electorateSize
status = !quorumMet ? EXPIRED
       : (approve * approvalDenominator >= approvalNumerator * (approve+reject)) ? APPROVED : REJECTED
```

Integer cross-multiplication throughout - no floating point, fully reproducible.
`EXPIRED` (quorum never reached) and `REJECTED` (quorum reached, majority did not) are
deliberately distinct outcomes, matching the brief's own scenario list.

**Execute** re-verifies staleness (`reference_state_digest` still matches current
state) before doing anything, then calls the *exact same* `ReferenceService.publishDirect()`/
`policyDirect()` methods the delegated-publisher path already uses and already
validates (evidence staleness, constitutional floors, protected fields) - governance
never reimplements that business logic, it only decides *whether* those methods may run
for a gated community. A proposal that crosses a constitutional floor still gets
refused at execute time even after unanimous approval
(`aProposalCrossingTheConstitutionalFloorCannotBeExecutedEvenUnanimouslyApproved`) -
"no importa que obtenga 100% de votos" holds by construction, by reuse, not by a
governance-side reimplementation of the floor check.

Staleness is a **computed fact, not a persisted status transition**: `execute()`
deliberately never writes a `VOID_STALE` status before throwing, because this service
runs as a real `@Transactional` Spring bean in production and a write immediately
followed by that same method's exception would be rolled back with it. Instead,
`proposal()`'s response carries a live-computed `stale` boolean; the persisted `status`
stays exactly what the vote itself decided (`APPROVED`), never silently rewritten by an
execution attempt that happened to fail.

## Immutability chain

proposal → electorate snapshot → votes → result → execution, exactly as required:
`ordinary_proposal` (frozen payload/digests, narrow `UPDATE(status,closed_at,executed_at)`
only) → `ordinary_proposal_electorate` (fully immutable) → `ordinary_vote` (fully
immutable) → the deterministic tally (never persisted as anything but the outcome of
that same immutable data) → `ordinary_proposal_execution` (fully immutable, unique per
proposal - a second `execute()` call is a 409, not a second effect). A later change to
`community_governance_member` or to `ordinary_governance_policy` never rewrites a
proposal already created against the old roster/policy version, because both are
copied/referenced by id+version at freeze time, never re-read live except at execute()'s
own staleness check (which only ever *blocks* a stale execution, never silently adapts
the vote to new rules).

## Privacy

Votes are individually attributed by design (`ordinary_vote.voter_id`), not secret
ballots - this is explicit auditability ("reconstruir quién podía votar, quién votó"),
not anonymity. What stays private: nothing about this feature exposes Participant
Independence's identity-continuity data during a vote - `community_governance_member`/
`ordinary_proposal_electorate`/`ordinary_vote` reference plain STIR `user_id`s only,
never a `ParticipantIndependenceService` lookup, never a `clusterRef`. Electorate
membership and identity-assurance clustering are two completely separate STIR-owned
concepts that never cross-reference each other's tables.

## Deliberately not built

- **A generic election/ballot framework or DSL.** One policy shape (v0.1), two
  proposal types (the two the brief named), no configurable vote-type registry.
- **Vote delegation, weighted voting, ranked choice.** One member, one vote, three
  choices (`APPROVE`/`REJECT`/`ABSTAIN`).
- **Automatic policy versioning/governance-of-governance.** The voting policy itself
  is set directly by `stir.governance.manage`, not itself voted on - avoiding infinite
  regress, matching "no framework electoral universal."
- **Cluster-aware relationship-diversity gating changing existing reason codes.** See
  `PARTICIPANT_INDEPENDENCE.md`'s hardening section - additive new reason codes, the
  original ones are untouched.

## Validation

Backend: `mvn verify` - `OrdinaryGovernancePostgresTest` (16 tests): full lifecycle
approve→execute→published-exact-reference, quorum-not-met→EXPIRED, quorum-met-no-
majority→REJECTED, double vote, user removed/added around an open vote, cross-tenant
RLS isolation, double execution, two proposals can't target the same reference
proposal, publisher bypass blocked once enabled (and unaffected when disabled),
platform SuperAdmin cannot vote or execute, constitutional-floor rejection surviving
unanimous approval, staleness after a concurrent policy edit, and electorate/vote
reconstruction.

HTTP E2E (`stir-main/scripts/ordinary_governance_e2e.py`): disabled-by-default direct
publish unaffected, publisher bypass blocked once enabled, permission/tenant
boundaries on every new endpoint, SuperAdmin rejected, double vote rejected, a real
vote reaching quorum and majority over a real HTTP-created electorate. The real
time-boxed deadline is not fast-forwarded live (same structural limitation as every
other same-day/real-clock scenario in this codebase) - that arithmetic is proven
directly against PostgreSQL with a backdated `voting_closes_at` instead.

Browser E2E (`stir-main/scripts/ordinary_governance_browser.py`): enabling governance,
building an electorate, saving a voting policy, starting a vote on a real reference
proposal, and two independent voter sessions casting real votes with the tally
updating live, and a blocked direct-publish attempt - all through the rendered UI, no
page errors. Caught and fixed two real bugs before this reached that state: (1) the
electorate "add member" form's submit button is `disabled` while its own request is in
flight, so three rapid-fire adds without waiting silently dropped one - fixed in the
test, not the product, since a real human clicking one at a time never hits this; (2) a
genuine product bug - `OrdinaryGovernancePanel` was nested inside `{publisher&&...}` in
`references.jsx`, meaning an electorate member without `stir.references.publish` could
never even see the voting panel, let alone vote - fixed by rendering it outside that
gate (its own management sections stay `canManage`-gated internally).
