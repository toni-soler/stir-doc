# Multi-source value evidence: LISTING, WANTED, COMMUNITY_SEED

## Five concepts, never collapsed into one

`COMMUNITY_VALUE_REFERENCES.md` already separated observation, reference and
agreed value. This increment adds two more real sources of evidence and one
deliberately non-market source, and keeps every one of them distinguishable
end to end - in storage, in the evidence math, and on screen:

| Source | What it represents | What it is not |
|---|---|---|
| `AGREEMENT` | An accepted, bilateral commercial term | Delivery, settlement, ledger commitment |
| `LISTING` | What a person is **offering** and under what conditions | "What the market accepts" |
| `WANTED` | What a person **seeks** and is willing to propose | An executed exchange |
| `PROPOSAL` | An in-flight negotiation offer (pre-existing; never itself counted - see "Economic lineage" below) | An agreement |
| `COMMUNITY_SEED` | A governed, normative initial orientation adopted by the community | Market evidence of any kind |

An expressed intention (LISTING/WANTED) is not an executed transaction
(AGREEMENT), and a community's own initial convention (COMMUNITY_SEED) is not
either. `EvidenceAnalysis`'s AGREEMENT/bilateral median is never blended with
LISTING or WANTED numbers into one figure - each source is its own,
separately computed, separately labeled bucket (see "Reference policy"
below), and a Community Seed is reported next to that evidence, never mixed
into it.

## LISTING/WANTED reuse the existing `Listing` entity

No new entity. `org.stir.listing.Listing` already had a `direction`
(`OFFER`/`WANTED`); this increment adds five nullable/opt-in fields mirroring
`Offer`'s own indicative-price shape exactly: `indicativeAmount`,
`indicativeQuantity`, `indicativeUnitLabel`, `indicativeUnitRef`,
`shareReferenceObservation`. This is the **owner's own ask**, never "what the
market accepts" - the same distinction `COMMUNITY_VALUE_REFERENCES.md` already
draws between an accepted Agreement and a reference.

**A later edit never rewrites an earlier observation.** `ListingRevision`
(new, mirrors `AgreementSnapshot`'s own style: immutable, JCS-canonical,
SHA-256-digested, monotonically versioned) is frozen on every `Listing`
create/update by `ListingRevisionService.freeze()`. A `reference_observation`
row's `source_id` points at the **revision's** id, never the mutable
`Listing.id` directly - editing the indicative price later creates a new
revision and a new observation; the first observation's own amount stays
exactly what it was when it was recorded.

`ListingEvidenceAdapter` (new, `org.stir.reference` package) is the **only**
crossing point from the listing bounded context into the reference bounded
context - the listing-side equivalent of the existing
`ReferenceAcceptanceAdapter`. It records an observation only when
`referenceDefinitionId` + `indicativeAmount` + `indicativeQuantity` are all
present, **regardless of consent** - having the datum and having permission
to use it stay separate questions, exactly as `CONSENT_RETENTION.md`
established for AGREEMENT. Consent capture is threaded through the same
`ConsentService.capture()`, now generalized with an explicit `purpose`
parameter (`ConsentService.LISTING_PURPOSE =
"LISTING_EVIDENCE_CONTRIBUTION"`, distinct from
`AGREEMENT_PURPOSE = "REFERENCE_EVIDENCE_CONTRIBUTION"`) - a LISTING/WANTED
never reuses an Agreement's bilateral consent semantics, because a listing's
consent is the owner's own, **unilateral** decision; there is no counterparty
to ask.

## Economic lineage: one process, never several independent voices

A single economic chain - LISTING → PROPOSAL → COUNTERPROPOSAL → AGREEMENT -
must never become four "independent voices" about value. `reference_observation`
gained a nullable `economic_lineage_id` column: the originating `Listing.id`,
threaded through the LISTING/WANTED observation itself and through any
PROPOSAL/AGREEMENT observation that descends from that same listing (via the
pre-existing `negotiation.listingId`). No new UUID generation was needed - the
listing's own id already is the natural correlation key for everything that
happens to it. `COMMUNITY_SEED` never has a lineage (`NULL`) - it is not an
economic process at all.

`EvidenceAnalysis.analyze()` groups **included** observations by
`economicLineageId` (or a synthetic `"no-lineage:"+id` key when null,
guaranteeing every observation is still counted somewhere) and reports
`economicLineageCount` and `maximumLineageShare` alongside the pre-existing
participant/pair concentration figures. A new exclusion-adjacent signal,
`LINEAGE_CONCENTRATED`, fires when `concentrationChecksRequired` is true and
one lineage's share of the bucket exceeds the same `maximumParticipantShare`
threshold the policy already defines - deliberately reusing that existing
number rather than inventing a second, arbitrary concentration parameter for
lineages. `ReferencePanel`'s "why" explainer shows this count with a plain-
language line: several observations from the same listing, negotiation or
agreement count as one economic chain, never as separate independent voices.

Note the one thing this does **not** prevent by itself: PROPOSAL-sourced
observations are excluded from the AGREEMENT bucket already
(`SOURCE_NOT_AGREEMENT`, pre-existing), so a listing → proposal → agreement
chain never actually produces three simultaneous "votes" in the same bucket
today regardless of lineage grouping - lineage grouping is what makes that
fact *provable and explainable* (one lineage, one origin `Listing.id`, N
observations across sources) rather than true only by the accident of which
buckets happen to accept which sources.

## Reference policy: separate buckets, never one blended weighting

`reference_policy` gained two ordinary booleans, `listing_source_enabled` and
`wanted_source_enabled`, both **false by default** - every existing
definition keeps its exact current AGREEMENT-only behavior until a publisher
or an approved ordinary-governance vote explicitly opts in.
`EvidenceAnalysis.Policy` gained `eligibleSource` (default `"AGREEMENT"`) and
`requiresCounterparty` (default `true`), each via a secondary constructor
preserving every prior call site's exact arity, plus a new
`Policy.forSource(...)` factory for a unilateral (LISTING/WANTED) bucket.

When a source is enabled, `ReferenceService.snapshot()` runs a **separate**
`EvidenceAnalysis.analyze()` call for it and folds the result into
`summary.listingEvidence`/`summary.wantedEvidence` as its own nested
sub-object (own status, own median/IQR when sufficient, own participant/
lineage counts) - never blended into the top-level AGREEMENT median. This is
a deliberate refusal to invent a mathematical weighting scheme for mixing
heterogeneous sources; the brief's own instruction was explicit separation,
not a new formula. `snapshot()` also always computes `sourceBreakdown` - raw
per-source counts (`AGREEMENT: 18`, `LISTING: 11`, `WANTED: 7`, ...),
**unconditionally**, regardless of any source's eligibility - so a publisher
or a curious member can always see how many raw observations of each kind
exist, without that count ever being presented as "N independent voices."

**A unilateral bucket still needs a real counterparty on nothing and a real
participant floor on everything.** `requiresCounterparty=false` skips the
`MISSING_COUNTERPARTY` check, the relationship/pair-based independence
checks (`REPEATED_RELATIONSHIP`, `INSUFFICIENT_INDEPENDENT_RELATIONSHIPS`,
etc. - meaningless with only one party per observation), and participant `b`
entirely. It does **not** skip the participant-*cluster*-based checks
(`INSUFFICIENT_INDEPENDENT_PARTICIPANTS`, `HIGH_INDEPENDENT_PARTICIPANT_CONCENTRATION`)
- those stay meaningful for a single-party source (many listings from the
same related cluster must not simulate diversity either), and they are
enforced against the exact same constitutional
`minimumParticipantFloor`/`minimumObservationFloor` the stored policy already
carries for the AGREEMENT bucket - **not** a separately relaxed floor for
LISTING/WANTED. Widening the input surface never weakens the provenance/
independence floor every observation has to pass.

Exclusion reasons gained two general-purpose codes reused across sources
rather than one new code per source: `SOURCE_NOT_ELIGIBLE` (the bucket's
`eligibleSource` does not match, used for every non-AGREEMENT bucket;
`SOURCE_NOT_AGREEMENT` is kept, byte-for-byte, as the AGREEMENT bucket's own
historical name for the same idea) and `NO_CONSENT` (the unilateral
counterpart to `NO_BILATERAL_CONSENT`).

## Community Seed: governed orientation, never a superuser's convenience

A Community Seed is **not** a market observation - it never touches
`reference_observation`, never has an `economic_lineage_id`, and is never
counted in `sourceBreakdown`. It is a separate, versioned, immutable table
(`stir.community_seed`) carrying exactly what a normative claim needs to be
honest about what it is: `kind` (`VALUE`/`BAND`/`QUALITATIVE`, same
vocabulary as a reference proposal), `lower_value`/`upper_value`,
`rationale`, `basis` (the governance act that adopted it - "Founding
assembly", "2026 general meeting"), `valid_from`/`valid_until`, and a foreign
key to the exact `ordinary_proposal` that approved it. A later seed is a new
`version`; the previous one is never rewritten (`community_seed` has no
UPDATE path at all, only INSERT).

**There is no delegated-publisher path for a seed at all** - unlike every
other publish/policy mutation in this bounded context
(`publishDirect()`/`policyDirect()`), which have a direct path for a
publisher acting without ordinary governance enabled, seed creation has
exactly one writer:
`ReferenceService.insertSeedDirect()`, package-private, callable only from
`OrdinaryGovernanceService.execute()`'s approved-proposal dispatch. A
platform SuperAdmin cannot create, vote on, or execute a seed proposal by
virtue of being SuperAdmin - `requireCommunityAuthority()` runs at the
controller and the service exactly as it already does for every other
governance mutation (`ORDINARY_GOVERNANCE.md`,
`GOVERNANCE_CAPTURE_THREAT_MODEL.md`). This is the strictest gate in the
codebase for a reason: an initial orientation is precisely the kind of claim
that is cheapest to fabricate and most tempting to impose unilaterally, so it
gets no shortcut whatsoever, not even the one every other ordinary mutation
still has.

`OrdinaryGovernanceService.isStale()` treats `COMMUNITY_SEED_PUBLICATION` as
never stale - unlike `PUBLISH_REFERENCE`/`REFERENCE_POLICY_CHANGE`, whose
staleness digest checks whether the evidence/policy state they were frozen
against has since drifted, a seed proposal's payload is fully self-contained
(a rationale and a range, not a snapshot of live evidence), so there is
nothing mutable it could drift against.

### Seed ≠ permanent truth (open SPEC GAP, by design)

`snapshot()` reports `communitySeed.supersededByRealEvidence` as a **live,
honestly-scoped** boolean: `true` exactly when the AGREEMENT bucket's own
status is `SUFFICIENT_DATA`. This is explicitly **not** an automatic
invalidation formula - the brief asked not to impose an arbitrary social rule
for exactly when a seed should stop being shown or stop influencing anything,
and none exists in this codebase. What this increment does guarantee: the
UI never hides the fact that real evidence now exists, and a seed is never
silently presented as if it were that evidence. What remains an open
**SPEC GAP**: whether/how a superseded seed should eventually stop being
shown at all, versus staying visible as historical context forever, versus
requiring a fresh governance vote to retire it explicitly. Do not improvise a
rule past this; escalate before picking one.

## Market Integrity: a new, cheaper-to-fabricate surface

LISTING and WANTED are cheaper to fabricate than AGREEMENT - publishing an
ad costs nothing, unlike getting a counterparty to actually accept a term.
`MarketIntegrityService.signal()`'s valid-code set gained seven new,
human-raised-only codes (still `SIGNAL ≠ FINDING` - no automated fraud
scoring anywhere in this codebase): `LISTING_SPAM`, `REPEATED_RELISTING`,
`COORDINATED_LISTING_OR_WANTED`, `TEMPORAL_BURST`, `LINEAGE_MANIPULATION`,
`SELECTIVE_CONSENT_PATTERN`, `SEED_ARTIFICIAL_ORIENTATION`. A case raised
against a LISTING/WANTED observation blocks its eligibility and its
anonymization exactly the same way an AGREEMENT-sourced case already does -
no new code path, the existing `finalExclusions`/retention-hold machinery
already generalizes because it keys on the observation, not the source.

## Reconstruction trail

Every observation - AGREEMENT, LISTING, WANTED alike - retains enough to
reconstruct source → source version/snapshot → consent → independence →
integrity → calculation → decision:

- **Source/version**: `source` + `source_id` (the immutable
  `ListingRevision`/`AgreementSnapshot` id, never the mutable parent id) +
  `economic_lineage_id` (the correlating origin, never itself mutable).
- **Consent**: `reference_consent`, purpose-tagged
  (`LISTING_EVIDENCE_CONTRIBUTION` vs `REFERENCE_EVIDENCE_CONTRIBUTION`),
  per party.
- **Independence**: the existing Participant Independence projection, applied
  identically regardless of source.
- **Integrity**: the existing `market_integrity_case` join, applied
  identically regardless of source.
- **Calculation/decision**: `EvidenceAnalysis`'s per-bucket `method`/`filters`
  (`SOURCE_MEDIAN_IQR_V1` for a non-AGREEMENT bucket, `AGREEMENTS_MEDIAN_IQR_V1`
  kept unchanged for AGREEMENT), reasons, and thresholds - unchanged
  mechanism, now also reported per source bucket.

`GET .../references/{id}/observations` (private, publisher-only, unchanged
permission gate) now also returns `economic_lineage_id` per row, so a
publisher can trace which raw observations share an origin without needing
to cross-reference the listing/negotiation tables by hand.

## UX

`ReferencePanel` shows, right below the existing AGREEMENT summary and never
implying that these numbers add into one voice count:

- `sourceBreakdown` - raw counts per source, always shown.
- `listingEvidence`/`wantedEvidence` - each source's own status, count,
  participant/lineage counts and median (when sufficient), only when that
  policy's source is enabled.
- `communitySeed` - kind, range, version, validity window, and the
  `supersededByRealEvidence` notice when applicable.
- Inside the existing "why" explainer: `economicLineageCount` with a
  plain-language line explaining that several observations from one
  economic chain count once, not several times.

`PolicyForm` (the direct-publisher policy editor) gained the two
`listingSourceEnabled`/`wantedSourceEnabled` checkboxes, reading and
resubmitting them explicitly on every save - see "A regression this
increment found and fixed" below for why that explicit round-trip matters.
`OrdinaryGovernancePanel`'s policy-change proposal form gained the same two
checkboxes for the governed path. A new seed-proposal form
(`govProposeSeed`) sits next to the existing "start a vote" actions, with an
explicit plain-language notice that a seed is a governed decision, never
market evidence, shown before the form fields themselves.

`ListingEditor` gained the four indicative-price fields plus the consent
checkbox, shown only once a comparable reference definition is selected, with
a plain-language hint that this is the owner's own ask, not a guarantee of
what the market will accept.

### A regression this increment found and fixed

`PolicyForm`'s submitted payload is deserialized into
`ReferenceController.PolicyRequest`, a Java record whose canonical
constructor includes `listingSourceEnabled`/`wantedSourceEnabled` as
primitive `boolean`s. Jackson deserializes a JSON body missing those keys by
defaulting missing primitives to `false` - so before this fix, `PolicyForm`
(which never included them) would have silently **disabled both sources on
every single unrelated policy edit** through the direct-publisher path (for
example, just widening the freshness window), even after they had been
correctly enabled through governance. The fix: `PolicyForm`'s `initial()` now
reads `policy.listing_source_enabled`/`wanted_source_enabled` into its own
form state, and its submit always resends them. Covered by
`multi_source_value_evidence_browser.py`: an unrelated field edit through the
rendered policy form, verified after a full page reload, leaves both
checkboxes checked.

## Extension boundary: revalidated, unchanged

Per `COMMUNITY_EXTENSION_GUIDE.md`'s existing boundary
(`distribution → STIR`, never the reverse), this increment adds nothing an
extension can use to bypass provenance, Consent/Retention, Market Integrity,
or Ordinary Governance. Nothing new is exposed to an extension: no SQL
credential, no privileged service account, no direct-write path to
`reference_observation`/`community_seed`. If a future extension genuinely
needs to propose a new evidence source of its own, it would need an explicit,
governed contract - the same "no plugin API built speculatively" restraint
`COMMUNITY_EXTENSION_GUIDE.md` already states, not something this increment
builds ahead of a real second consumer needing it.

## Deliberately not built

- **A mathematical weighting formula across sources.** Each source stays its
  own explicit, separately labeled bucket - see "Reference policy" above.
- **An automatic seed-supersession rule.** See "Seed ≠ permanent truth"
  above - an open SPEC GAP, not a silently assumed default.
- **A generic evidence-source plugin API for extensions.** See "Extension
  boundary" above.
- **Automated fraud scoring for the new manipulation surface.** Signals stay
  human-raised only, same `SIGNAL ≠ FINDING` discipline as
  `MARKET_INTEGRITY.md`.
- **A relaxed independence/observation floor for unilateral sources.** They
  are held to the exact same constitutional floor as AGREEMENT - see
  "Reference policy" above.

## Validation

Backend: `mvn verify` - `MultiSourceValueEvidencePostgresTest` (13 tests):
a valid LISTING/WANTED contributes only once its policy is enabled and
consent is given; LISTING and WANTED contribute as separate buckets, never
blended; editing a listing never rewrites its earlier observation; a
listing → proposal → agreement chain shares one economic lineage
(`economic_lineage_id` correlates all three); multiple accounts from the same
independence cluster do not inflate LISTING diversity, held to the same
constitutional participant floor as AGREEMENT; withdrawal removes future
eligibility without deleting history; a Community Seed proposal approved
through a real ordinary-governance vote is executed and recorded; a seed
proposal rejected for lack of quorum, or with quorum but insufficient
approval, is never applied and cannot be executed; platform SuperAdmin cannot
propose, vote on, or execute a seed; a second seed is a new version that
never rewrites the first; double execution of an approved seed is blocked;
tenant isolation holds for listings, seeds and lineage. Zero regressions
across the pre-existing 181 tests.

HTTP E2E (`stir-main/scripts/multi_source_value_evidence_e2e.py`): the same
scenarios against a real running backend - a real LISTING and WANTED with
indicative prices and per-party consent; editing a listing preserving its
earlier observation; a listing → proposal → agreement chain sharing one
economic lineage; tenant isolation; SuperAdmin excluded from proposing or
voting on a seed; double voting rejected; a real vote over the real
electorate reaching quorum and majority. Seed close/execute/replay/
double-execution/new-version mechanics are proven directly against
PostgreSQL with a backdated `voting_closes_at` in the Postgres test above,
not over real HTTP - the same real-clock limitation
`ordinary_governance_e2e.py` already established (`votingWindowHours` has a
hard one-real-hour floor).

Browser E2E (`stir-main/scripts/multi_source_value_evidence_browser.py`): a
real LISTING and a real WANTED published with their own indicative price and
explicit consent through the rendered listing form; the source/lineage
breakdown rendering with LISTING and WANTED kept separate from AGREEMENT; the
`PolicyForm` regression above, reproduced and verified fixed through the
rendered form; a Community Seed proposed through a real ordinary-governance
vote - all through the actually rendered UI, no JavaScript page errors. Note
on ordering: `snapshot()`/`sourceBreakdown` cache their result once per
calendar day for a given definition, the first time they are computed - a
same-day observation is structurally invisible to a snapshot already cached
earlier the same day, even after backdating it for the daily-cutoff proof
below. This script therefore never visits `/stir/references` for its
definition until after both listings exist and their observations are
backdated, matching the same real-clock discipline
`community_references_e2e.py`'s own daily-reconstruction proof already
established.
