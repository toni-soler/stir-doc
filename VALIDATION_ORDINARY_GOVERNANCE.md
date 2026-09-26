# Ordinary Community Governance — change and validation record

Full design in `ORDINARY_GOVERNANCE.md`; the two Participant Independence
hardenings bundled into this same increment are documented in
`PARTICIPANT_INDEPENDENCE.md`'s own Hardening sections. Branch
`claude/ordinary-governance-mvp` across `stir-backend`, `stir-frontend`,
`stir-main`, `stir-doc`, `stir-workspace`.

## Inspection phase (required before any design decision)

Before writing code: verified there is no existing STIR/idax-core mechanism
to enumerate "every user holding permission P" (idax-shell's own admin
`GET /roles`/`/roles/{id}/permissions`/`/roles/users` endpoints are gated on
`system.roles.read`, an IDAX admin permission no ordinary community member
holds, and no STIR code has ever called them) - resolved by making the
electorate a wholly STIR-owned roster, never derived from IDAX permissions,
avoiding a new cross-service credential entirely. Re-read Seven Keys'
proposal→signature→activation pattern in full
(`SevenKeysService.java`/`V6__market_integrity_governance.sql`/
`V7__enforce_seven_seats.sql`) to reuse its freeze-at-creation, staleness-
recheck-at-activation, and append-only-history conventions exactly for
ordinary governance instead of inventing a parallel pattern.

## Exact changed files

**stir-backend**

```text
src/main/resources/db/migration-stir/V11__ordinary_governance.sql       (new)
src/main/resources/db/migration-stir/V12__independence_coverage_policy.sql  (new)
src/main/resources/generated/stir/permission-catalog.generated.json    (regenerated - see below)
config/permission-catalog.json                                         (+stir.governance.manage, +stir.governance.vote)
src/main/java/org/stir/reference/OrdinaryGovernanceService.java        (new)
src/main/java/org/stir/reference/OrdinaryGovernanceController.java     (new)
src/main/java/org/stir/reference/ReferenceService.java                 (+requireDirectMutationAllowed gate, +publishDirect/policyDirect)
src/main/java/org/stir/reference/ReferenceController.java              (PolicyRequest +minimumIndependenceCoveragePercent)
src/main/java/org/stir/reference/EvidenceAnalysis.java                 (+cluster-aware relationship metrics, +coverage-floor reason)
src/test/java/org/stir/reference/OrdinaryGovernancePostgresTest.java   (new, 16 tests)
src/test/java/org/stir/reference/EvidenceAnalysisTest.java             (+3 tests)
src/test/java/org/stir/reference/SevenKeysPostgresTest.java            (PolicyRequest call-site update only)
```

**stir-frontend**

```text
src/api.js               (+ordinaryGovernanceApi)
src/ordinary-governance.jsx  (new - OrdinaryGovernancePanel, ProposalCard)
src/references.jsx       (+OrdinaryGovernancePanel, moved outside the publisher-only gate)
src/locales.json          (+52 keys x 12 locales = 490 keys each)
```

**stir-main**

```text
scripts/ordinary_governance_e2e.py       (new)
scripts/ordinary_governance_browser.py   (new)
```

**stir-doc**

```text
ORDINARY_GOVERNANCE.md                    (new - full design)
VALIDATION_ORDINARY_GOVERNANCE.md         (this file)
PARTICIPANT_INDEPENDENCE.md               (+2 hardening sections, deferred-item updated)
```

**stir-workspace**: `CLAUDE.md` (new governance capability referenced).

No osTRIS/idax-core file was changed. No generator/template for the
IDAX-side permission catalog was changed - **`permission-catalog.generated.json`
is STIR's own generated artifact** (`python scripts/generate-permissions.py`
regenerates it from `config/permission-catalog.json`); it is git-tracked and
must be regenerated and committed whenever the source catalog changes - this
was missed on the first pass and caught by the HTTP E2E script failing with a
403 on every governance-manage call, tracing back through container logs to
confirm the running image's registered catalog only had the original 17
entries, not the 2 new ones. Fixed by running the generator and rebuilding.

## Design constraints honored explicitly

- **No `approveProposal(id)`.** The only path to `APPROVED`/`REJECTED`/
  `EXPIRED` is `OrdinaryGovernanceService.lazyClose()`'s deterministic tally
  over the frozen electorate and recorded votes - there is no method
  anywhere in this codebase that sets that status directly.
- **Execution reuses, never reimplements, existing validation.** `execute()`
  calls `ReferenceService.publishDirect()`/`policyDirect()` - the exact same
  methods (minus their own `requireCommunityAuthority`/gate checks, done by
  the caller) the delegated-publisher path already validates against
  (evidence staleness, constitutional floors, protected fields). A
  constitutional-floor-crossing proposal is refused at execute time even
  after unanimous approval, by construction, not by a governance-side
  reimplementation.
- **SuperAdmin/Guardian/Seven-Keys-seat grants no vote by itself.** The
  electorate is a wholly separate, explicit roster; `requireCommunityAuthority`
  gates every mutation the same as everywhere else in this bounded context.
- **Backward compatible by construction.** `ordinary_governance_enabled`
  defaults `false`; every pre-existing test/script/direct-publish flow is
  unaffected until a community explicitly opts in - proven by
  `withoutGovernanceEnabledDirectPublishStillWorksExactlyAsBefore` and by
  every pre-existing regression script staying green.
- **Immutability chain preserved**: proposal → electorate snapshot → votes →
  result → execution, each layer copying/referencing the layer below by
  id/version, never live-re-reading a roster or policy that could have moved
  on - staleness only ever blocks a future execution, never rewrites a past
  vote (`policyChangedAfterProposalCreationVoidsExecutionAsStale`).

## Gates

| Gate | Result and evidence |
|---|---|
| Backend verify | PASS: `mvn verify`, 156 tests (137 pre-existing unaffected + 16 OrdinaryGovernancePostgresTest + 3 EvidenceAnalysisTest hardening), 0 failures |
| Frontend tests | PASS: `npm test`, 38 tests, 0 failures |
| Frontend build | PASS: `npm run build` |
| i18n | PASS: `npm run i18n:validate`, 12 locales, 490 keys each (52 new `gov*` keys - 2 renamed to avoid colliding with Seven Keys' existing `govProposals`/`govSubmitProposal`) |
| Existing E2E regression | PASS: all 4 pre-existing HTTP scripts (`community_references_e2e.py`, `market_integrity_e2e.py`, `community_value_governance_e2e.py`, `participant_independence_e2e.py`) and all 4 pre-existing browser scripts - zero regressions from the `ReferenceService`/`EvidenceAnalysis` changes |
| **New: HTTP E2E** | PASS: `ordinary_governance_e2e.py` - disabled-by-default direct publish unaffected, publisher bypass blocked once enabled, permission/tenant boundaries, SuperAdmin rejected, double vote rejected, real quorum+majority over a real electorate |
| **New: browser E2E** | PASS: `ordinary_governance_browser.py` - governance enabled, electorate built, policy saved, real vote started on a real reference proposal, two independent voter sessions voting live, direct-publish bypass blocked - caught and fixed one test-only race and one real product bug (panel wrongly nested inside a publisher-only gate) before reaching green |
| Clean Docker | PASS: `docker compose build stir stir-ui` |

The real time-boxed voting deadline is never fast-forwarded live in HTTP/
browser E2E (same structural limitation as every other real-clock scenario
in this codebase); quorum/majority/expiry arithmetic across a closed vote is
proven directly against PostgreSQL with a backdated `voting_closes_at`
(`OrdinaryGovernancePostgresTest`).

An unrelated infrastructure incident during this increment's E2E work: a
Flyway checksum mismatch (from editing `V11` mid-development) led to wiping
the local dev Postgres volume, which in turn exposed a stale `proxy`
container stuck in `Created` state after the rebuild - neither was a STIR
code defect; both were fixed by recreating the affected containers.

## Explicitly not attempted

Per `ORDINARY_GOVERNANCE.md`'s "deliberately not built" section: a generic
election/ballot framework; vote delegation/weighted/ranked-choice voting;
governance-of-the-voting-policy-itself (the policy is set directly by
`stir.governance.manage`, not voted on). Per `PARTICIPANT_INDEPENDENCE.md`'s
updated hardening section: cluster-identity aggregation independent of a
shared hub account remains open.
