# Participant independence / identity integrity

## The gap this closes

STIR could already measure *account diversity* (`EvidenceAnalysis`'s distinct
participant count, pair concentration, repeated-relationship checks) but had
no way to know whether diverse-looking accounts represented independent
economic actors. `EvidenceAnalysis.java` shipped with a hardcoded,
never-computed placeholder (`identityAssurance:
"ACCOUNTS_ONLY_RELATED_ACCOUNTS_UNKNOWN"`) and `COMMUNITY_VALUE_REFERENCES.md`
listed "Verified related-account grouping and independent-person counts" as
deliberately deferred. This increment makes that field real, without
inventing independence STIR cannot prove.

**The governing principle, repeated everywhere in this design:** ten accounts
do not automatically mean ten independent participants. STIR corrects
diversity *down* when it has positive evidence of relatedness; it never
corrects diversity *up* by treating "unknown" as "independent." When STIR
cannot know, it says exactly that - `INDEPENDENCE_UNKNOWN` - and never blocks
a valid Agreement over it.

## What already existed in osTRIS (verified in source, not assumed)

osTRIS already has the full identity-continuity primitive this needed -
`Participant`, `RiskSubject`, `IdentityAssuranceClaim`,
`IdentityContinuityDecision`, all at
`stir-main/vendor/ostris/backend/.../V1__ostris_core.sql` and
`IdentityContinuityService.java`/`IdentityContinuityController.java`:

- `ostris.risk_subject`: an opaque, community-scoped correlation anchor - no
  PII, not a global person ID (`IDENTITY_ASSURANCE.md`: "must not use DNI,
  passport, email, phone, tax ID or their unsalted hash").
- `ostris.participant.risk_subject_id`: set only when a continuity decision
  is `CONFIRMED`; cleared for `CONTESTED`/`REJECTED`.
- `ostris.identity_continuity_decision`: immutable, sequence-stamped
  (`community_sequence`, via `ostris.community.next_sequence` - this **is**
  the staleness/versioning mechanism, not a separate class), status
  `CONFIRMED`/`CONTESTED`/`REJECTED`.
- `GET /api/ostris/identity/communities/{community}/participants/{participant}/continuity`
  (`OSTRIS_IDENTITY_CONTINUITY_READ_PRIVATE`): returns the latest effective
  decision, or a 422 `CONTINUITY_NOT_FOUND` when none was ever recorded.
  Already privacy-shaped: only an opaque `riskSubjectId` when `CONFIRMED`,
  never PII.

**osTRIS never affirmatively certifies independence.** It only ever
`CONFIRMED`s, `CONTESTED`s or `REJECTED`s one specific relatedness claim
between a participant and a candidate `RiskSubject`. `REJECTED` is not an
independence certificate - it means only "this specific claim was rejected."
STIR therefore only ever narrows diversity down (confirmed relatedness
collapses accounts into one identity unit) and never widens it (no status
from osTRIS ever adds confidence that unrelated-looking accounts truly are
independent).

## The real gap: STIR-side credentialing, not an osTRIS-side one

`OstrisClient` has exactly one calling convention: relay the *inbound
request's own bearer token*. But `ReferenceService.snapshot()` computes
evidence lazily, once per definition per UTC day, cached and reused for
whoever happens to read it first - most readers hold `stir.references.read`,
not osTRIS's own `OSTRIS_IDENTITY_CONTINUITY_READ_PRIVATE`. Relaying an
arbitrary reader's token to ask osTRIS a private identity question would
fail for most triggers, or - worse - make evidence quality depend on which
reader happened to load the page first that day.

**Resolved (explicit user decision) as: a persisted, minimal, versioned
projection, refreshed only by an explicit publisher action, never called
live from `snapshot()`.** Two hard constraints shaped the design:

1. **The publisher only triggers; osTRIS decides.** `ParticipantIndependenceService.refresh()`
   persists *exactly* what osTRIS's own protocol response says - a
   publisher cannot pass in a status or a cluster; they can only ask again.
2. **Minimize what STIR stores.** The raw osTRIS `risk_subject_id` is never
   persisted. Only `SHA-256(tenant_id:community_id:risk_subject_id)`
   (`clusterRef`) is stored - enough to recognize two participants share a
   cluster (equality-comparable), never enough to reconstruct osTRIS's own
   correlation key from a copy of STIR's database alone.

A new STIR→osTRIS service credential (bypassing the relay-the-caller's-token
model entirely) was explicitly deferred as a future automation option, not
built now - it would introduce a new cross-service trust boundary that
deserves its own normative decision, not something to bundle into this
increment.

## Data model

`stir.participant_independence_projection` (Flyway V10, STIR schema) -
tenant/user-scoped, RLS-forced like every other STIR table:

| Column | Meaning |
|---|---|
| `user_id` | STIR's own actor id (matches `reference_observation.participant_a/b`) |
| `status` | `NO_OSTRIS_BINDING` \| `INDEPENDENCE_UNKNOWN` \| `IDENTITY_CONTINUITY_PENDING` \| `RELATED_CONTINUITY` |
| `cluster_ref` | SHA-256 digest, only when `RELATED_CONTINUITY` |
| `community_sequence` | osTRIS's own sequence this reflects, for historical reconstruction |
| `fetched_at` / `fetched_by` | when and which publisher triggered this refresh |

Not append-only by design - unlike `market_governance_event` or
`reference_snapshot`, a projection is a *current-state cache* of osTRIS's
latest answer, not an evidence/governance history record, so `UPDATE`
(idempotent upsert) is legitimate here. The upsert is monotonic: a response
with an older `community_sequence` than what is already stored never
overwrites it (`staleOrOutOfOrderOstrisResponseNeverDowngradesAFresherProjection`).

`ParticipantIndependenceService.refresh(user, targetUserId)`:

1. `ReferenceService.requireCommunityAuthority(user)` - same boundary as
   every other sensitive mutation in this bounded context
   (`GOVERNANCE_CAPTURE_THREAT_MODEL.md`); being platform SuperAdmin does
   not grant this either.
2. No `participant_economic_binding` row for this user → `NO_OSTRIS_BINDING`,
   osTRIS is never called.
3. Otherwise, call osTRIS's real private continuity endpoint (relaying the
   *caller's own token* - the publisher must hold
   `OSTRIS_IDENTITY_CONTINUITY_READ_PRIVATE` themselves, same relay-only
   model as every other `OstrisClient` call):
   - `CONFIRMED` → `RELATED_CONTINUITY` + `clusterRef`.
   - `CONTESTED` → `IDENTITY_CONTINUITY_PENDING`.
   - `REJECTED` or `CONTINUITY_NOT_FOUND` (422) → `INDEPENDENCE_UNKNOWN`.

New endpoints, `ReferenceController`, both under the existing
`stir.references.publish`-gated, publisher-only pattern (same trust level as
`observations()`/`evidence-manifest()`, not a new one):

- `POST .../references/participants/{userId}/independence-refresh`
- `GET .../references/participants/{userId}/independence`

## Where it slots into evidence

`EvidenceAnalysis.analyze(...)` takes a new optional
`Map<UUID,Independence> independence` parameter (backward compatible -
existing 4/6-arg overloads default to `Map.of()`, so every pre-existing
caller/test is byte-identical). `ReferenceService.snapshot()` reads the
projection via `ParticipantIndependenceService.forUsers(tenant, accounts,
policy.freshnessDays, cutoff)` - a plain `SELECT`, **never** a live osTRIS
call from inside `snapshot()`. A projection older than the reference's own
`freshnessDays` policy is treated as absent, same as if it never existed.

Two independent effects, both additive (never weaken or replace an existing
check, only catch what raw counting missed):

1. **Per-observation exclusion.** If both counterparties resolve to the
   *same* `clusterRef`, the observation is excluded with reason
   `RELATED_PARTICIPANT_CLUSTER` - the same tier as the existing
   `MISSING_COUNTERPARTY` (literal same-account) check, generalized to
   "same identity, different accounts." The Agreement itself is never
   rewritten, only excluded from that day's eligible evidence - exactly
   like every other exclusion reason.
2. **Aggregate correction.** Among the accounts behind *included*
   observations, confirmed-related accounts collapse into one identity
   unit (`clusterKey()` - a confirmed cluster's digest, or the account's own
   id if unrelated/unknown). New reason codes, gated on
   `policy.independenceChecksRequired()` **and** non-zero independence
   coverage (so a reference nobody has ever run a refresh on behaves
   exactly as before - zero behavioral change without new data):
   - `INSUFFICIENT_INDEPENDENT_PARTICIPANTS` - adjusted count below
     `minimumParticipants`, even though raw `participants.size()` alone
     would have passed.
   - `HIGH_INDEPENDENT_PARTICIPANT_CONCENTRATION` - a cluster's combined
     observation share exceeds `maximumParticipantShare`, even when no
     single raw account individually crosses that floor (a sybil spreading
     dominance across two same-cluster accounts).

New public summary fields (suppressed to `null` for small/insufficient
cohorts, same privacy rule as `median`/`participantCount`):
`identityAssurance` (now a real computed value, not a hardcoded string),
`independenceAssuranceCoveragePercent`, `adjustedIndependentParticipantCount`,
`relatedAccountClusters`, `unknownIndependenceAccountCount`.

## Historicity

The private evidence blob (`reference_snapshot.evidence_json`, surfaced via
`evidence-manifest`) now also records `independenceProjections`: exactly
which projection (status + osTRIS `community_sequence`) this frozen snapshot
used, per account. A later `IdentityContinuityDecision` can never silently
rewrite why an old snapshot looked the way it did - reconstruction replays
what was recorded, not today's state. If a new decision changes future
eligibility, that shows up only in a later day's new snapshot/version,
exactly like every other policy or integrity-finding change in this system.

## What is deliberately not built here

- **A STIR→osTRIS service credential** for fully automatic, reader-triggered
  refresh - explicitly deferred (see above); the persisted-projection design
  needs no new cross-service trust boundary.
- **Cluster-aware relationship-diversity counting** (`relationshipCount`/
  `INSUFFICIENT_INDEPENDENT_RELATIONSHIPS`/`REPEATED_RELATIONSHIP` still
  count raw account pairs, not cluster pairs). A sybil cluster represented
  through several *different* raw account pairs against different
  counterparties is not yet caught by the relationship-diversity checks,
  only by the participant-count/concentration ones. A real fix needs a
  cluster-pair aggregation pass; flagged here rather than silently left.
- **`IdentityAssuranceClaim` (KYC-level) revocation tracking** - the private
  continuity endpoint this integration uses answers relatedness, not
  assurance-claim lifecycle; `IDENTITY_ASSURANCE_REVOKED` from the original
  brief's example reason-code list has no real data source wired yet.
- **Cross-community exposure** - independence data is strictly
  tenant/community scoped, same RLS boundary as everything else in this
  schema; a relation confirmed in one community is never visible from
  another tenant's session (`ParticipantIndependencePostgresTest.tenantADoesNotAcquireAuthorityOverTenantBMutationsThroughThisGuard`-style
  isolation, verified again for this endpoint specifically).

## Privacy minimization, explicitly checked

- No raw osTRIS `risk_subject_id` persisted - only a one-way digest.
- The actor who can see "these two accounts are related" (a publisher,
  gated on `stir.references.publish`) does not see *why*, nor any name,
  document, or provider detail - osTRIS's own private endpoint already
  returns nothing more than the opaque `riskSubjectId`.
- Platform SuperAdmin exclusion is preserved exactly as
  `GOVERNANCE_CAPTURE_THREAT_MODEL.md` requires:
  `requireCommunityAuthority` gates `refresh()` the same as every other
  mutation. The pre-existing residual gap (private `observations()`/
  `evidence-manifest()` reads still ride idax-core's superuser permission
  bypass) is unchanged and not widened by this increment - the new
  `independence`/`independence-refresh` endpoints sit at exactly the same
  `stir.references.publish` trust tier as those, not a new or broader one.

## Validation

Backend: `mvn verify` - `EvidenceAnalysisTest` (pure function, 6 new tests:
same-cluster exclusion, different-cluster non-exclusion, raw-diversity-at-
floor corrected down, split-concentration-across-two-cluster-members,
mixed known/unknown coverage, honest-placeholder-without-data) and
`ParticipantIndependencePostgresTest` (13 tests: no-binding, real
`CONTINUITY_NOT_FOUND`/`CONTESTED`/`REJECTED` mapping, SuperAdmin rejection,
monotonic upsert, idempotency, staleness-past-policy, per-observation
exclusion, aggregate correction, mixed coverage, historicity).

HTTP E2E (real osTRIS, not a mock):
`stir-main/scripts/participant_independence_e2e.py` - no-binding, a real
osTRIS `CONTINUITY_NOT_FOUND` response, a real osTRIS `CONFIRMED` decision
(risk_subject seeded via SQL only because osTRIS itself has no HTTP creation
endpoint for it - the decision itself always goes through osTRIS's real,
permissioned HTTP contract), permission/tenant boundaries on both new
endpoints, raw Agreement history surviving the same-day cutoff. Per-reason-
code and aggregate-arithmetic proof is *not* attempted live over HTTP - same
structural reason `VALIDATION_COMMUNITY_VALUE_GOVERNANCE.md` already
documents for every other same-day scenario (`snapshot()`'s `< cutoff` SQL
filter makes a same-day observation invisible to any evidence computation
for the rest of that real calendar day) - it is proven instead at the
PostgreSQL and pure-function levels above.

Frontend: `EvidenceBreakdown` shows a refresh badge per observation
participant (`IndependenceBadge`); `ReferencePanel`'s evidence section shows
coverage/adjusted-count/cluster/unknown counts and the identity-assurance
label. `npm test`/`npm run i18n:validate` both green, 12 locales, 20 new
`ref*` keys each.
