# Multi-Source Value Evidence — change and validation record

Full design in `MULTI_SOURCE_VALUE_EVIDENCE.md`. Branch
`claude/multi-source-value-evidence-mvp` across `stir-backend`,
`stir-frontend`, `stir-main`, `stir-doc`, `stir-workspace`. Built directly on
top of the Consent/Retention MVP already on `main` (`stir-backend 0bb3a4a`,
`stir-frontend 2835cae`, `stir-main 06499f1`, `stir-doc 046cc1f`).

## Exact changed files

**stir-backend**

```text
src/main/resources/db/migration-stir/V15__multi_source_value_evidence.sql  (new)
src/main/java/org/stir/listing/Listing.java                        (+5 indicative-price/consent fields)
src/main/java/org/stir/listing/ListingRequest.java                 (+5 fields, 2 secondary constructors preserved)
src/main/java/org/stir/listing/ListingView.java                    (+5 fields)
src/main/java/org/stir/listing/ListingRevision.java                (new - immutable snapshot, mirrors AgreementSnapshot)
src/main/java/org/stir/listing/ListingRevisionRepository.java      (new)
src/main/java/org/stir/listing/ListingRevisionService.java         (new - freeze())
src/main/java/org/stir/listing/ListingService.java                 (calls evidence.recorded(revisions.freeze(listing)) on create/update)
src/main/java/org/stir/reference/ListingEvidenceAdapter.java       (new - the only listing->reference crossing point)
src/main/java/org/stir/reference/ReferenceService.java             (record() +economicLineageId overload; snapshot() +sourceBreakdown/listingEvidence/wantedEvidence/communitySeed; policyDirect() +2 columns; +seedHistory()/currentSeed()/insertSeedDirect(); observations() +economic_lineage_id)
src/main/java/org/stir/reference/ReferenceAcceptanceAdapter.java    (passes negotiation.listingId as lineage; consent.capture() +purpose)
src/main/java/org/stir/reference/EvidenceAnalysis.java             (Policy +eligibleSource/+requiresCounterparty; Observation +economicLineageId; lineage grouping + LINEAGE_CONCENTRATED; unilateral-source gating throughout analyze())
src/main/java/org/stir/reference/ConsentService.java                (capture() +purpose param; +AGREEMENT_PURPOSE/+LISTING_PURPOSE constants)
src/main/java/org/stir/reference/ReferenceController.java          (PolicyRequest +2 booleans, constructors preserved; +GET seed-history)
src/main/java/org/stir/reference/OrdinaryGovernanceService.java    (+SeedRequest, +proposeCommunitySeed(); execute()/isStale() +COMMUNITY_SEED_PUBLICATION; proposePolicyChange() payload +2 fields)
src/main/java/org/stir/reference/OrdinaryGovernanceController.java (+POST proposals/{definitionId}/seed)
src/main/java/org/stir/reference/MarketIntegrityService.java       (+7 signal codes)
src/test/java/org/stir/listing/ListingServiceTest.java              (mocks widened constructor)
src/test/java/org/stir/reference/ConsentRetentionPostgresTest.java  (consent.capture() call sites +AGREEMENT_PURPOSE)
src/test/java/org/stir/reference/MultiSourceValueEvidencePostgresTest.java (new, 13 tests)
```

**stir-frontend**

```text
src/catalog-client.js    (listingPayload() +5 fields when referenceDefinitionId is set)
src/extension.jsx        (ListingEditor +4 indicative-price fields +consent checkbox)
src/references.jsx       (+SourceEvidenceSummary; ReferencePanel +sourceBreakdown/listingEvidence/wantedEvidence/communitySeed/economicLineageCount; PolicyForm +2 checkboxes, regression fix; SIGNAL_CODES +7)
src/ordinary-governance.jsx (+seedForm state/UI; policy-change form +2 checkboxes)
src/api.js                (+proposeCommunitySeed, +seedHistory)
tests/references.test.mjs (dynamic-suffix lists extended for new reason/source/signal codes)
src/locales.json          (+38 keys x 12 locales = 552 keys each)
```

**stir-main**

```text
scripts/multi_source_value_evidence_e2e.py       (new)
scripts/multi_source_value_evidence_browser.py   (new)
```

**stir-doc**

```text
MULTI_SOURCE_VALUE_EVIDENCE.md                (new - full design)
VALIDATION_MULTI_SOURCE_VALUE_EVIDENCE.md     (this file)
COMMUNITY_VALUE_REFERENCES.md                 (+SOURCE_NOT_ELIGIBLE/NO_CONSENT/LINEAGE_CONCENTRATED, deliberately-deferred update)
COMMUNITY_EXTENSION_GUIDE.md                  (revalidation note: no new extension entry point)
```

**stir-workspace**: `CLAUDE.md` (new capability section, SPEC GAP #4 added).

No osTRIS/idax-core file was changed. No generator/template affected (this
domain has no generator involvement - `gen-idax-legacy` is IDAX-specific,
not part of the STIR bounded context).

## Design constraints honored explicitly

- **LISTING ≠ WANTED ≠ PROPOSAL ≠ AGREEMENT ≠ COMMUNITY_SEED, never
  collapsed.** Each is its own explicit `source` value; `sourceBreakdown`
  reports raw counts per source without ever implying they are
  equally-weighted votes; LISTING/WANTED get their own separate
  `EvidenceAnalysis` buckets, never blended into the AGREEMENT median.
- **A later listing edit never rewrites an earlier observation.**
  `editingAListingNeverRewritesAnEarlierObservation` (Postgres) and the HTTP
  E2E's own `select amount ... order by observed_at` check both prove two
  distinct rows survive an edit, in original order, with original values.
- **One economic chain never becomes several independent voices.**
  `listingProposalAgreementChainSharesOneLineageNotThreeVoices` proves a
  single `economic_lineage_id` correlates the LISTING, PROPOSAL and
  AGREEMENT observations descending from one listing.
- **Cluster-aware independence applies to unilateral sources too.**
  `multipleAccountsOfTheSameClusterDoNotInflateListingDiversity` - two
  clustered accounts collapse to one adjusted participant, held to the exact
  same constitutional `minimumParticipantFloor=6` as AGREEMENT (not a
  relaxed floor for LISTING).
- **Consent stays purpose-specific.** LISTING/WANTED never reuse
  `AGREEMENT_PURPOSE`; `withdrawalRemovesFutureEligibilityWithoutDeletingHistory`
  proves withdrawal excludes future eligibility without touching history,
  same discipline `CONSENT_RETENTION.md` already established.
- **Community Seed has zero delegated-publisher path.**
  `platformSuperAdminCannotProposeVoteOrExecuteASeed` proves SuperAdmin is
  excluded at every step (propose/vote/execute), not just one of them.
  `seedRejectedForLackOfQuorumNeverAppliesAndCannotBeExecuted` and
  `seedQuorumMetButNoMajorityIsRejected` prove a seed only ever applies
  through a genuinely passing vote.
- **A seed is a new version, never a rewrite.**
  `aNewSeedIsANewVersionNeverARewriteOfTheOlderOne` proves the first seed's
  `lower_value` stays byte-identical after a second seed is approved.
- **Replay/double execution blocked.**
  `doubleExecutionOfAnApprovedSeedIsBlocked`, same idempotency discipline as
  every other approved-proposal execution path in this codebase.
- **Tenant isolation holds for listings, seeds and lineage.**
  `tenantIsolationListingSeedAndLineageNeverCrossTenants`.

## A real regression found and fixed during validation

`references.jsx`'s `PolicyForm` (the direct-publisher policy editor) never
included `listingSourceEnabled`/`wantedSourceEnabled` in its submitted
payload. Because `ReferenceController.PolicyRequest` is a Java record with
these as primitive `boolean`s, Jackson silently defaulted both to `false` on
deserialization whenever they were missing from the JSON body - meaning
**any** unrelated policy edit through that rendered form (for example
widening the freshness window) would have silently disabled LISTING/WANTED
evidence, even after they had been correctly enabled through governance.
Found while writing `multi_source_value_evidence_browser.py` against the
real running stack (the `PolicyForm` never reflected the current enabled
state at all before the fix, since `initial()` didn't read the two fields
either). Fixed in `stir-frontend` commit `77c988c`; reproduced and verified
fixed by the browser script itself (edit an unrelated field, save, reload
the page, confirm both checkboxes stay checked).

A second, smaller gap found the same way: the private
`GET .../references/{id}/observations` reconstruction-trail endpoint never
selected `economic_lineage_id`, so a publisher had no way to see which raw
observations shared an origin without cross-referencing the listing table by
hand. Fixed in `stir-backend` commit `53876b3`.

## Gates

| Gate | Result and evidence |
|---|---|
| Backend verify | PASS: `mvn verify`, 194 tests (181 pre-existing unaffected + 13 `MultiSourceValueEvidencePostgresTest`), 0 failures, migrations V1-V15 applied across every Testcontainers run |
| Frontend tests | PASS: `npm test`, 40 tests, 0 failures |
| Frontend build | PASS: `npm run build` |
| i18n | PASS: `npm run i18n:validate`, 12 locales, 552 keys each (38 new keys) |
| Existing E2E regression | PASS: `community_references_e2e.py`, `ordinary_governance_e2e.py`, `market_integrity_e2e.py`, `consent_retention_e2e.py`, `ordinary_governance_browser.py`, `consent_retention_browser.py` - zero regressions from the `ReferenceService`/`EvidenceAnalysis`/`ConsentService`/`OrdinaryGovernanceService` changes |
| **New: HTTP E2E** | PASS: `multi_source_value_evidence_e2e.py` - real LISTING/WANTED with indicative prices and consent, history-preserving edits, one shared economic lineage across a listing->proposal->agreement chain, tenant isolation, SuperAdmin excluded from a seed proposal/vote, double vote rejected, a real vote reaching quorum and majority |
| **New: browser E2E** | PASS: `multi_source_value_evidence_browser.py` - a real LISTING and WANTED published through the rendered form, the source/lineage breakdown rendering, the PolicyForm regression reproduced and verified fixed, a Community Seed proposed through the rendered governance panel |
| Clean Docker | PASS: `python scripts/deploy.py` (build, recreate, health check, smoke check) |

## Real testing frictions, worked around the same way established precedent already does

**Same-day snapshot caching bit the browser script on the first draft.**
`ReferenceService.snapshot()` caches its result per calendar day, the first
time it is computed for a definition (`reference_snapshot` keyed by
`cutoff=start of day`) - visiting `/stir/references` to toggle the
listing/wanted policy checkboxes *before* creating any listings froze an
empty snapshot for the rest of that day, and no amount of backdating the
observations afterward could un-freeze it (same "don't design a test that
expects same-day eligibility to change live" property
`VALIDATION_COMMUNITY_VALUE_GOVERNANCE.md` already documented). Fixed by
reordering the script: enable both sources via real HTTP setup, create both
listings through the rendered UI, backdate their observations, and only then
visit `/stir/references` for the first time - so that first-ever snapshot
computation already reflects the final state. The `PolicyForm` regression
proof itself doesn't depend on the snapshot cache at all (`currentPolicy()`
reads the policy table directly), so it runs safely after that first visit.

**`source_id` is the `ListingRevision` id, not the `Listing` id - by
design**, exactly mirroring how an AGREEMENT observation's `source_id` is
the `AgreementSnapshot` id, not the `Agreement` id. The first draft of the
HTTP E2E script incorrectly assumed `source_id == listing['id']`; fixed to
correlate via `economic_lineage_id` instead, which is the field that exists
specifically for this purpose.

**A permission gap in the browser script's own fixture, not a product bug**:
`GET /members/{communityId}` requires `stir.governance.vote`, not just
`stir.governance.manage` - the manager role in the first draft granted only
the latter, so the electorate list silently 403'd and stayed empty (swallowed
by the panel's own `.catch(() => {})`). Fixed by granting both permissions,
matching `ordinary_governance_browser.py`'s already-correct manager role.

**Real time-boxed vote, same real-clock limitation as every other governance
script.** `votingWindowHours` has a hard one-real-hour floor, so no HTTP
script can observe a seed vote's `close()`/`execute()` transition. The HTTP
E2E script proves proposal creation, SuperAdmin exclusion (propose and vote),
double-vote rejection, and tally correctness only; close/execute/quorum/
majority/replay/double-execution/new-version mechanics are proven directly
against PostgreSQL with a backdated `voting_closes_at` in
`MultiSourceValueEvidencePostgresTest`, exactly the discipline
`ordinary_governance_e2e.py` already established.

**Constitutional floor uniformity required fixing a test, not the product.**
`multipleAccountsOfTheSameClusterDoNotInflateListingDiversity` originally
used only 2 clustered observations, expecting the bucket to reach
`SUFFICIENT_DATA` - but the same `minimumParticipantFloor=6` that applies to
AGREEMENT applies identically to a LISTING-only bucket (both read the same
stored `reference_policy` numbers), so 2 observations can never be
sufficient regardless of independence. Fixed by adding 5 unrelated
single-listing observations to reach the real floor - this is the product
behaving exactly as designed, not a bug the test needed to route around.

## Explicitly not attempted

Per `MULTI_SOURCE_VALUE_EVIDENCE.md`'s "Deliberately not built" section: a
mathematical weighting formula across sources; an automatic seed-supersession
rule (open SPEC GAP); a generic evidence-source plugin API for extensions;
automated fraud scoring for the new manipulation surface; a relaxed
independence/observation floor for unilateral sources.
