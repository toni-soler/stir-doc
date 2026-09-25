# Community Value Governance — change and validation record

Before this increment, Community Value References had a fully working
propose/publish cycle in the UI, but nothing else was reachable except by
`curl`/direct SQL: no way to see which raw observations existed, which of
them counted and why, no way to tune a reference's own operational policy,
and no UI for Market Integrity at all (signal a concern, review it, decide).
This increment closes that gap without redesigning anything Codex already
built and validated (`COMMUNITY_VALUE_REFERENCES.md`,
`MARKET_INTEGRITY.md`, `VALIDATION_COMMUNITY_VALUE_REFERENCES.md`,
`VALIDATION_MARKET_INTEGRITY.md` were read in full before writing any code).
All four repositories use local branch `claude/community-value-governance-mvp`.

## Exact changed files

**stir-backend**

```text
src/main/java/org/stir/reference/ReferenceService.java       (+observations(), +evidenceManifest())
src/main/java/org/stir/reference/ReferenceController.java    (+GET .../observations, +GET .../evidence-manifest, +SuperAdmin rejection)
src/main/java/org/stir/reference/MarketIntegrityController.java  (+SuperAdmin rejection)
src/test/java/org/stir/reference/ReferencePostgresTest.java  (+2 tests)
```

**stir-frontend**

```text
src/references.jsx           (+EvidenceBreakdown, +PolicyForm, +IntegrityPanel, propose-time
                               deviation hint, reconstruction trail on proposals/history)
src/reference-comparison.js  (+proposalDeviation())
src/api.js                   (referenceApi: +observations, +evidenceManifest, +policy;
                               +integrityApi)
src/locales.json             (+51 keys x 12 locales = 418 keys each)
tests/references.test.mjs    (+2 tests: proposalDeviation, extended locale coverage)
```

**stir-main**

```text
scripts/community_value_governance_e2e.py       (new)
scripts/community_value_governance_browser.py   (new)
```

**stir-doc**

```text
COMMUNITY_VALUE_REFERENCES.md          (evidence-manifest/observations now have endpoints;
                                         policy editor shipped; SuperAdmin exclusion documented)
GOVERNANCE_CAPTURE_THREAT_MODEL.md     (idax-core superuser-bypass finding and its scoped fix)
VALIDATION_COMMUNITY_VALUE_GOVERNANCE.md   (this file)
```

No osTRIS, IDAX Core, .NET or private book repository was changed. No generator
or template was changed and no generated file was edited.

## A hard architectural constraint this increment had to design around

`ReferenceService.snapshot()`'s SQL only ever considers observations strictly
before today's UTC cutoff (`observed_at < cutoff`) — by design, the
anti-live-differencing rule `MARKET_INTEGRITY.md` already documents. A raw
observation created "just now" is not merely excluded with a reason from that
day's evidence-manifest; it is invisible to the query that builds it, for the
rest of that real calendar day. This means no live HTTP or browser run — on
a single real day — can reach `SUFFICIENT_DATA`, show a real median, or
exercise a specific per-observation exclusion reason code
(`NO_BILATERAL_CONSENT`, `OUTSIDE_WINDOW`, ...). Those scenarios were already
proven correct with explicitly backdated timestamps, at two levels:
`EvidenceAnalysisTest.java` (pure unit level, already comprehensive — repeated
A↔B, concentration, real vs. fabricated outliers, sybil-looking accounts,
FINAL exclusion) and `ReferencePostgresTest` (through the real service and
schema). This increment's own new backend test,
`observationsAndEvidenceManifestExposeRawVsEligibleOnlyToPublishers`, backdates
observations the same way to prove `observations()`/`evidenceManifest()`
classify correctly. The two new E2E scripts prove what a live same-day run
*can* honestly prove instead: `observations()` has no such cutoff and always
reflects the complete, undeleted raw history; `evidence-manifest` never
differs within the same day no matter how many more deals happen; and every
new endpoint's permission/tenant boundary holds. Neither script claims a
same-day reason-code demonstration it cannot actually produce.

## A real, current security finding — and its scoped fix

While verifying "a platform SuperAdmin gets no community-governance capability
by being SuperAdmin" (an explicit requirement for this increment), the
straightforward `@PreAuthorize("@permissionService.hasPermission('stir.
references.publish')")` check turned out **not** to enforce that: idax-core's
`PermissionService.hasPermission(user, permission)` returns `true`
unconditionally for `user.isSuperuser()`, for every permission string, in
every tenant — confirmed by decompiling the vendored
`idax-core-0.4.0.jar` and empirically (a superuser token with no tenant role
read a tenant's reference definition, 200 not 403). This is a platform-wide
property of every `stir.*`-gated endpoint in STIR, not something this or any
prior STIR increment introduced, and patching the vendored dependency is out
of scope for this session. `ReferenceController` and `MarketIntegrityController`
now reject `authentication.principal.superuser` explicitly in their shared
`tenant()` `@ModelAttribute`, closing the gap for every endpoint on both
controllers (existing ones - `propose`, `publish`, `policy` - and the new
ones alike). `SevenKeysController` needed no equivalent change: its real
protection is the Ed25519 signature requirement, independent of the HTTP
permission layer, so this same bypass cannot forge a seat's vote there. Full
writeup, including the residual exposure on any *other* `stir.*` controller
until it adds the same explicit guard, is in
`GOVERNANCE_CAPTURE_THREAT_MODEL.md`.

## Gates

| Gate | Result and evidence |
|---|---|
| Backend verify | PASS: `mvn verify`, 120 tests, 0 failures/errors (118 pre-existing + 2 new) |
| Frontend tests | PASS: `npm test`, 35 tests (33 pre-existing + 2 new), 0 failures |
| Frontend build | PASS: `npm run build` |
| i18n | PASS: `npm run i18n:validate`, 12 locales, 418 keys each (51 new `ref*` keys) |
| Existing E2E regression | PASS: `community_references_e2e.py`, `community_references_browser.py`, `market_integrity_e2e.py`, `governance_ui_browser.py` all still green |
| **New: HTTP E2E** | PASS: `community_value_governance_e2e.py` — raw observation history survives regardless of same-day manifest caching, anti-live-differencing, platform SuperAdmin explicitly excluded, ordinary policy governance inside constitutional floors with floor(400)/protected-field(409) rejection, market integrity SIGNAL→UNDER_REVIEW→FINAL and →DISMISSED by a genuinely different publisher, tenant isolation |
| **New: browser E2E** | PASS: `community_value_governance_browser.py` — evidence breakdown, reference policy configuration, v1 propose/publish, a real Agreement growing the raw observation count, market integrity signal→review→FINAL across two independent publisher sessions, v2 publish with v1 unchanged — all through the rendered UI, no page errors |
| Clean Docker | PASS: `docker compose build --no-cache stir stir-ui` |

The HTTP/browser fixtures create disposable local tenants only. Agent-started
`stir`/`stir-ui`/`shell`/`postgres`/`ostris`/`ledger` containers were stopped
after verification.

## Explicitly not attempted

Per `GOVERNANCE_CAPTURE_THREAT_MODEL.md`'s standing SPEC GAPs: no automated
identity-cluster claim beyond `ACCOUNTS_ONLY_RELATED_ACCOUNTS_UNKNOWN`, and
`REPLACE_CONTROLLER` stays fail-closed. Neither was touched by this increment.
A UI for the *protected* constitutional fields (independence/concentration
checks, the floors themselves) was deliberately not built — those change only
through a signed 7-of-7 Seven Keys amendment, never through the ordinary
policy form.
