# Consent/Retention — change and validation record

Full design in `CONSENT_RETENTION.md`. Branch `claude/consent-retention-mvp`
across `stir-backend`, `stir-frontend`, `stir-main`, `stir-doc`,
`stir-workspace`. Built directly on top of the Community Extension
Integration MVP already on `main` (`stir-backend 41e6d7a`,
`stir-frontend f5884b3`, `stir-main c211c5d`, `stir-doc 23a8f9c`,
`stir-workspace adeb9d8`).

## Exact changed files

**stir-backend**

```text
src/main/resources/db/migration-stir/V14__consent_retention.sql   (new)
src/main/java/org/stir/reference/ConsentService.java              (new)
src/main/java/org/stir/reference/ConsentController.java           (new)
src/main/java/org/stir/reference/RetentionService.java            (new)
src/main/java/org/stir/reference/RetentionController.java         (new)
src/main/java/org/stir/reference/ReferenceService.java            (record() returns id; withdrawn-set exclusion; requireDirectMutationAllowed widened to package-private)
src/main/java/org/stir/reference/ReferenceAcceptanceAdapter.java  (captures each party's own consent record)
src/main/java/org/stir/reference/OrdinaryGovernanceService.java   (+RETENTION_POLICY_CHANGE proposal type)
src/main/java/org/stir/reference/OrdinaryGovernanceController.java (+propose-retention-policy-change endpoint)
config/permission-catalog.json                                    (+stir.consent.manage, +stir.retention.manage)
src/main/resources/generated/stir/permission-catalog.generated.json (regenerated)
src/test/java/org/stir/reference/ConsentRetentionPostgresTest.java (new, 16 tests)
src/test/java/org/stir/reference/OrdinaryGovernancePostgresTest.java (constructor call site updated only)
```

**stir-frontend**

```text
src/consent-retention.jsx  (new - MyConsentPanel, RetentionPanel)
src/api.js                 (+consentApi, +retentionApi, +proposeRetentionPolicyChange)
src/profile.jsx            (+MyConsentPanel in MyProfile)
src/references.jsx         (+RetentionPanel, publisher-gated like IntegrityPanel)
src/locales.json           (+24 keys x 12 locales = 514 keys each)
```

**stir-main**

```text
scripts/consent_retention_e2e.py       (new)
scripts/consent_retention_browser.py   (new)
```

**stir-doc**

```text
CONSENT_RETENTION.md                (new - full design)
VALIDATION_CONSENT_RETENTION.md     (this file)
COMMUNITY_VALUE_REFERENCES.md       (+CONSENT_WITHDRAWN exclusion reason, deliberately-deferred update)
```

**stir-workspace**: `CLAUDE.md` (new capability referenced, floor documented).

No osTRIS/idax-core file was changed. No generator/template affected.
`permission-catalog.generated.json` was regenerated (`python
scripts/generate-permissions.py`) and confirmed present before rebuilding
the Docker image - the exact gotcha caught and documented during Ordinary
Governance was checked proactively this time, not rediscovered.

## Design constraints honored explicitly

- **Four questions, never collapsed.** Having a datum (`reference_observation`
  always recorded), having permission to use it (`reference_consent`),
  still being allowed to retain it (`retention_policy`/`RetentionStatus`),
  and still being eligible as current evidence (`EvidenceAnalysis`'s live
  exclusion set) are four separately computed, separately named states -
  proven directly by `marketIntegrityCaseForcesRetentionHoldEvenPastWindow`
  (simultaneously `CONSENT_WITHDRAWN`-eligible-for-new-use-question and
  `MUST_BE_RETAINED` on the retention question, without contradiction).
- **No silent history rewrite.** Consent withdrawal never touches
  `aggregate_consent`, the Agreement, or any cached `reference_snapshot`
  (`cachedSnapshotSurvivesALaterWithdrawalUnchanged`). Retention's one real
  mutation (anonymization) is enforced at the trigger level to touch only
  `participant_a`/`participant_b`/`anonymized_at`/`anonymized_by`, nothing
  else, ever (`rawSqlCannotBypassTheAnonymizationOnlyTrigger`).
- **Market Integrity evidence cannot be erased via consent withdrawal.**
  `finalIntegrityFindingTakesPriorityOverWithdrawalReason`; a market-integrity
  case unconditionally blocks anonymization regardless of retention age
  (`marketIntegrityCaseForcesRetentionHoldEvenPastWindow`).
- **Platform SuperAdmin excluded**, same shared `requireCommunityAuthority`
  check as every other sensitive mutation in this bounded context
  (`platformSuperAdminCannotSetRetentionPolicyOrAnonymize`).
- **Retention floor is a code constant** (`RetentionService.MINIMUM_RETENTION_PERIOD_DAYS = 90`),
  deliberately not added to the Seven Keys constitution schema - see
  CONSENT_RETENTION.md's "Retention" section for why, and the SPEC GAP this
  leaves open if a future increment needs it constitutionally enforced.
- **Ordinary Governance does not silently freeze once enabled.** Before
  `RETENTION_POLICY_CHANGE` existed, enabling governance would have blocked
  direct retention-policy changes with no proposal path to replace them -
  closed in this same increment, proven by
  `ordinaryGovernanceCanChangeRetentionPolicyAndDirectChangeIsBlockedOnceEnabled`.

## Gates

| Gate | Result and evidence |
|---|---|
| Backend verify | PASS: `mvn verify`, 181 tests (165 pre-existing unaffected + 16 `ConsentRetentionPostgresTest`), 0 failures, migrations V1-V14 applied across every Testcontainers run |
| Frontend tests | PASS: `npm test`, 40 tests, 0 failures |
| Frontend build | PASS: `npm run build` |
| i18n | PASS: `npm run i18n:validate`, 12 locales, 514 keys each (24 new `consent*`/`retention*` keys) |
| Existing E2E regression | PASS: `community_references_e2e.py`, `ordinary_governance_e2e.py`, `market_integrity_e2e.py`, `community_references_browser.py` - zero regressions from the `ReferenceService`/`ReferenceAcceptanceAdapter`/`OrdinaryGovernanceService` changes |
| **New: HTTP E2E** | PASS: `consent_retention_e2e.py` - bilateral consent, one-sided decline, withdrawal exclusivity/idempotency, tenant isolation, retention floor, SuperAdmin exclusion, market-integrity hold blocking anonymization, anonymization preserving amount/history |
| **New: browser E2E** | PASS: `consent_retention_browser.py` - a real party withdrawing their own consent from the profile page with the impact notice shown first, and a publisher anonymizing an eligible observation from the rendered retention panel |
| Clean Docker | PASS: `docker compose up -d --build` |

## A real testing limitation, worked around the same way twice

`ReferenceService.snapshot()`'s daily UTC cutoff cache means the **first**
Agreement acceptance for a definition on a given day permanently caches
that day's evidence state - including for definitions created fresh inside
this same E2E run, whose very first `accept()` call triggers
`freezeContext()` → `view()` → `snapshot()` before there is any chance to
backdate an observation into it. This is the same "daily cut resists live
differencing" property `community_references_e2e.py` already works around
by only ever proving cache *stability*, never real-time recomputation
(`VALIDATION_COMMUNITY_VALUE_GOVERNANCE.md`). `consent_retention_e2e.py`
follows the identical discipline: the `CONSENT_WITHDRAWN` exclusion-reason
mechanism is proven directly against PostgreSQL, where `observed_at` is
controllable at insert time before any snapshot ever computes
(`ConsentRetentionPostgresTest.withdrawalAfterAcceptanceExcludesFromFutureSnapshotsWithDistinctReason`);
the HTTP/browser scripts prove everything else the daily cache doesn't
block - consent capture, withdrawal exclusivity and idempotency, tenant
isolation, and the retention/anonymization path, which has no such cache
constraint since it never calls `snapshot()` at all.

A second, real constraint the new `reject_reference_observation_mutation()`
trigger introduced: no ordinary role - not even `multitenant.sql()`'s own
real Postgres superuser connection - can `UPDATE observed_at` on
`reference_observation` anymore, by design. Backdating an observation for
E2E fixture purposes (there is no HTTP path to do this, and there should
never be one) now requires the fixture SQL to explicitly
`SET session_replication_role = 'replica'` first, itself only possible for
a genuine superuser connection - a stronger, more deliberate bypass than a
simple role reset, used only in test fixtures, never by the application
itself.

## Explicitly not attempted

Per `CONSENT_RETENTION.md`'s "Deliberately not built" section: a generic
legal/compliance engine; a manual governance-hold API (the only real basis
today is an existing market-integrity case); scheduled/automatic
anonymization; attachment/photo retention; a Seven-Keys-amendable retention
floor; a shared consent/retention primitive for extensions.
