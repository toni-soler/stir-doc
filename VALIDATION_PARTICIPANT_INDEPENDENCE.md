# Participant Independence / Identity Integrity — change and validation record

Full design in `PARTICIPANT_INDEPENDENCE.md`. This record covers the
increment itself: what changed, where, and how each gate was proven. Branch
`claude/participant-independence-mvp` across `stir-backend`, `stir-main`,
`stir-doc`, `stir-workspace`.

## Inspection phase (required before any design decision)

Before writing any code, osTRIS's actual identity infrastructure and STIR's
current account/evidence model were inspected directly in source (not
assumed). Finding: osTRIS already has the full primitive
(`Participant`/`RiskSubject`/`IdentityAssuranceClaim`/
`IdentityContinuityDecision`, immutable, sequence-stamped, already
privacy-shaped) - the real gap was narrower than expected: STIR-side
credentialing (how a lazily-triggered, cached-per-day `snapshot()`
computation could ask osTRIS's *private* continuity endpoint on behalf of
whichever reader happens to trigger it, most of whom lack that osTRIS
permission), not a missing osTRIS contract. That exact SPEC GAP was
presented to the user with three concrete resolution paths; the chosen one -
a persisted, minimal, versioned, publisher-triggered projection, never
called live from `snapshot()` - avoided introducing any new cross-service
trust boundary.

## Exact changed files

**stir-backend**

```text
src/main/resources/db/migration-stir/V10__participant_independence_projection.sql  (new)
src/main/java/org/stir/economic/OstrisClient.java              (+privateContinuity, +IdentityContinuityView)
src/main/java/org/stir/reference/ParticipantIndependenceService.java  (new)
src/main/java/org/stir/reference/ReferenceService.java          (+independence dependency, snapshot() wiring)
src/main/java/org/stir/reference/ReferenceController.java       (+independence-refresh, +independence)
src/main/java/org/stir/reference/EvidenceAnalysis.java          (+Independence record, +cluster-aware metrics/reasons)
src/test/java/org/stir/reference/EvidenceAnalysisTest.java      (+6 tests)
src/test/java/org/stir/reference/ParticipantIndependencePostgresTest.java  (new, 13 tests)
src/test/java/org/stir/reference/ReferencePostgresTest.java     (constructor call-site updates only)
src/test/java/org/stir/reference/SevenKeysPostgresTest.java     (constructor call-site update only)
```

**stir-frontend**

```text
src/api.js            (+independence, +refreshIndependence)
src/references.jsx    (+IndependenceBadge, evidence panel independence summary)
src/style.css          (+.stir-inline-action, +.stir-hint)
src/locales.json       (+20 keys x 12 locales = 438 keys each)
```

**stir-main**

```text
scripts/participant_independence_e2e.py  (new)
```

**stir-doc**

```text
PARTICIPANT_INDEPENDENCE.md              (new - full design)
VALIDATION_PARTICIPANT_INDEPENDENCE.md   (this file)
COMMUNITY_VALUE_REFERENCES.md            (deferred-item resolved, manipulation table row updated)
```

**stir-workspace**: `CLAUDE.md` (SPEC GAP #2 updated to "partially closed").

No osTRIS/idax-core file was changed - osTRIS's existing contract was
sufficient. No generator or template was changed.

## Design constraints honored explicitly

- **Never invent independence.** Every new reason code and metric only ever
  *narrows* diversity when osTRIS has confirmed relatedness; nothing ever
  *widens* it. Accounts with no data are `INDEPENDENCE_UNKNOWN`, counted as
  their own unit (same as before this increment), never merged and never
  assumed independent.
- **Never block a valid Agreement.** The per-observation exclusion affects
  eligible *evidence*, never the Agreement itself - the raw
  `reference_observation` row is untouched (proven explicitly:
  `confirmedSameClusterCounterpartyIsExcludedAsRelatedNotIndependentEvidence`
  asserts the raw row count is unchanged).
- **osTRIS stays the sole identity authority.** STIR persists only what
  osTRIS's own protocol response says, never authors or alters it
  (`refreshWithNoOstrisBindingRecordsThatExplicitlyWithoutCallingOstris`,
  `refreshOnRejectedIsUnknownNotAnIndependenceCertificate`).
- **Minimize sensitive data.** No raw `risk_subject_id` is stored - only a
  one-way digest (`refreshOnConfirmedRecordsClusterRefNotRawRiskSubjectId`
  asserts the stored value is neither the raw UUID nor derivable from it
  without the tenant/community context).
- **SuperAdmin exclusion preserved exactly**, not widened, not narrowed
  (`refreshRejectsPlatformSuperAdminSameGuardAsEveryOtherMutation`).
- **Historical reconstruction preserved.** A frozen snapshot records which
  `community_sequence` it used per account; a later decision cannot rewrite
  why an old snapshot looked the way it did.

## Gates

| Gate | Result and evidence |
|---|---|
| Backend verify | PASS: `mvn verify`, 137 tests (124 pre-existing unaffected + 6 EvidenceAnalysisTest + 13 ParticipantIndependencePostgresTest), 0 failures |
| Frontend tests | PASS: `npm test`, 35 tests, 0 failures |
| Frontend build | PASS: `npm run build` |
| i18n | PASS: `npm run i18n:validate`, 12 locales, 438 keys each (20 new `ref*` keys) |
| Existing E2E regression | PASS: `community_references_e2e.py`, `market_integrity_e2e.py`, `community_value_governance_e2e.py` (HTTP); `community_references_browser.py`, `governance_ui_browser.py`, `community_value_governance_browser.py` (browser) - all still green with the new `EvidenceAnalysis`/`ReferenceService` wiring in place |
| **New: HTTP E2E** | PASS: `participant_independence_e2e.py` - real osTRIS calls (not mocked): `NO_OSTRIS_BINDING`, real `CONTINUITY_NOT_FOUND` (422), a real `CONFIRMED` decision via osTRIS's own HTTP contract, permission/tenant boundaries on both new endpoints, raw Agreement history surviving the same-day evidence-manifest cutoff |
| Clean Docker | PASS: `docker compose build stir stir-ui` |

Per-reason-code and aggregate-arithmetic proof (raw diversity 6 → adjusted 5,
concentration split across two cluster members, mixed coverage) is proven at
the PostgreSQL and pure-function levels, not live over HTTP - the same
structural reason (`snapshot()`'s `observed_at < cutoff` SQL filter) already
documented in `VALIDATION_COMMUNITY_VALUE_GOVERNANCE.md` for every other
same-day scenario in this codebase. `PARTICIPANT_INDEPENDENCE.md`'s
Validation section lists exactly which test proves which scenario.

The HTTP/browser fixtures create disposable local tenants only; the one
`ostris.risk_subject` row seeded directly via SQL is a test fixture only
(osTRIS itself has no HTTP creation endpoint for that row by design - it is
an internal correlation anchor, never client-supplied). Agent-started
`stir`/`stir-ui`/`shell`/`postgres`/`ostris`/`ledger` containers were stopped
after verification.

## Explicitly not attempted

Per `PARTICIPANT_INDEPENDENCE.md`'s "deliberately not built" section: a
STIR→osTRIS service credential for fully automatic refresh; cluster-aware
relationship-diversity counting; `IdentityAssuranceClaim` revocation
tracking. None of these were silently skipped - each is named with the
reason it stayed out of scope.
