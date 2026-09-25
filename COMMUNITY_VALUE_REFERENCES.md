# Community value references — STIR v0.1

## Architectural diagnosis

The reviewed baseline is STIR 0.5.0-rc1, split into independent workspace,
backend, frontend, deployment and documentation repositories. Profiles are
tenant-scoped presence, not verified economic identities. Listings have
OFFER/WANTED direction but **no asking-price field**. Negotiations contain
alternating offers, optional quantity, quantity label, amount and unit reference.
Acceptance creates an Agreement and an immutable AgreementSnapshot in the same
transaction. Reads of private negotiations and agreements check both tenant and
party membership; PostgreSQL FORCE RLS enforces tenant isolation.

AgreementSnapshot → osTRIS EXCHANGE is already implemented, not future work.
TradeService relays the user's token, proposes the accepted amount, and sends the
existing RFC 8785 JCS/SHA-256 AgreementSnapshot digest as contractualMetadataDigest.
Client signatures and osTRIS commit determine economic completion. No goods,
statistical reference or fairness interpretation belongs to that protocol.

The implementation adds `org.stir.reference`, one migration (V5), an authenticated
frontend page and small Listing/Negotiation/Agreement adapters. There is no new
repository, service, runtime dependency or osTRIS change. The gate for extracting
this domain is a **second application besides STIR needing the same community
references**. Until then JDBC persistence, descriptive analysis and publication
remain inside STIR; acceptance is the explicit integration boundary.

## Five separate concepts

| Concept | Representation | Authority |
|---|---|---|
| Unit of account | Existing explicit marketplace community/unit binding | osTRIS unit rules |
| Observation | Immutable source-tagged evidence | What a STIR source actually recorded |
| Community reference | Immutable published version of an explicit proposal | A community's delegated publisher and recorded decision |
| Agreed value | Accepted Offer / AgreementSnapshot | The two parties |
| Committed entry | osTRIS EXCHANGE / Trade reconciliation | osTRIS authorization and commit |

Agreement evidence means **accepted commercial terms**, not delivery, settlement,
or ledger commitment. Its source is never relabeled COMMITTED. A community may
deliberately choose guidance that differs from the observed median. Proposals,
publication, statistical sufficiency and negotiation acceptance are separate
actions; deviation never blocks an agreement.

## Definition, quantity and identity

A definition identifies an immutable comparison scope: name, human description,
up to 20 bounded string attributes, positive quantity basis and exact quantity
unit. For example, `Bread / plain 500g loaf / {weight: 500g} / 1 loaf`, or
`Translation / specified language pair and complexity / 1000 words`.
Changing the comparable object creates another definition; this is deliberately
not a universal ontology, SKU registry or unit conversion engine.

Community and unit UUIDs come from the **existing explicit economic binding**.
Creating a definition requires that binding to exist. Tenant identity is never
used as community identity. `unit_ref` is the exact bound unit UUID string;
negotiation's free-text `proposedUnitRef` must match it for statistical comparison.
Missing or different units/quantities produce NOT_COMPARABLE, not inferred parity.
Normalized descriptive amount = accepted total amount × definition quantity basis
÷ agreed quantity. No fiat equivalence or promise of convertibility is introduced.

A listing can associate a definition. A negotiation freezes that association when
opened, so later listing edits cannot silently reclassify its historical offers.
Existing unassociated negotiations stay unassociated. Comparison uses the proposed
quantity and exact units, not the listing title or a guessed category.
When a linked offer form opens, STIR fills its quantity basis and exact unit
identifiers; either party can still change those fields. The agreement screen
shows the accepted value from its contractual snapshot next to the separate
reference context, with technical unit UUIDs tucked into an audit detail.

## Evidence and policy

The evidence schema distinguishes LISTING, WANTED, PROPOSAL, AGREEMENT and
COMMUNITY_SEED. v0.1 automatically records linked proposals and accepted agreements.
Listings have no numeric asking-price field, so this release does not manufacture
listing-price observations. WANTED/seed/import capture is reserved for explicit
future adapters. An initial convention is a **proposal/decision**, never a synthetic
accepted trade used to inflate sample size.

The only implemented descriptive source policy is AGREEMENT. Policies are
append-only versions: default 90-day window, at least 5 agreements, at least 6
distinct accounts, at most 40% of agreements involving any one account, and a
newest agreement no older than 30 days. Configuration can tighten these rules;
hard privacy floors disallow fewer than 5 observations or 6 accounts and more
than 50% participant concentration. No generic policy language is introduced.

The method `AGREEMENTS_MEDIAN_IQR_V1` reports the median and the observed-order
quartiles at floor((n−1)/4) and floor(3(n−1)/4). It does not extrapolate bands or
trim outliers. Decimal arithmetic normalizes at 8 places internally and displays
2 decimal places. A normative reference band is explicitly entered by people;
the observed interquartile range is separately labeled. Maximum participant and
pair shares expose concentration when the sample is publishable.

Each immutable snapshot contains the exact UTC window, method, filters, effective
policy identity/version and thresholds, state, reasons, descriptive fields and a
JCS/SHA-256 digest. A private evidence manifest stores included observation IDs
and the reason for each excluded input. Exclusion precedence is deterministic:
SOURCE_NOT_AGREEMENT, NO_BILATERAL_CONSENT, OUTSIDE_WINDOW, NOT_COMPARABLE,
MISSING_COUNTERPARTY. Since the Community Value Governance increment, this
manifest is reachable at `GET .../references/{id}/evidence-manifest`, and the
underlying raw rows (including `participant_a`/`participant_b`) at
`GET .../references/{id}/observations` - both gated on `stir.references.publish`,
never on ordinary read access, and never surfaced through the public snapshot.
A publisher reviewing why a specific accepted Agreement did or did not count
toward today's reference is the intended use; this is not a public evidence feed.

### Daily cuts, rebuilding and privacy

The cutoff is UTC midnight at the beginning of the current day. Observations
created today participate in tomorrow's calculation. The first calculation per
definition/day is frozen and reused; policy edits after that calculation take
effect on the next daily snapshot. Reading a reference can lazily create this
snapshot inside a transaction. There is no timer, external cache or refresh API
that lets users choose arbitrary subcohorts or time boundaries. Restarting STIR
reads identical stored bytes. Independent reconstruction uses the recorded
immutable evidence/policy/cutoff; do not overwrite a published snapshot.

Both the author of the **accepted head offer** and the accepting party must
explicitly opt in to aggregate statistics. Checkboxes default off. Consent on a
superseded offer does not transfer to the new author. Messages, terms, source IDs,
pair identities, participant IDs and individual observation values are not
exposed in community evidence responses. Private offers are not visible to a
publisher simply because that actor can publish references.

When a cohort fails any threshold, the public result is INSUFFICIENT_DATA with
reasons, but without amounts, exact counts, newest individual observation time
or concentration ratios. The UI intentionally cannot say “based on 2 agreements”
for a private small cohort. Successful cohorts show counts and timestamps.
This avoids an immediate single-agreement price disclosure. Daily reuse limits
live differencing; it is **not differential privacy** and does not defeat an
adversary with extensive outside knowledge or coordinated accounts. Do not
advertise mathematical anonymity or identity assurance that STIR does not have.

## Governance and publication

Permissions are separately grantable:

- `stir.references.read`: definitions, public aggregate evidence, proposals/history.
- `stir.references.propose`: definitions and immutable reference proposals.
- `stir.references.publish`: explicit publication and new policy versions.
- Existing `stir.agreements.read` plus party checks: private historical context.

Permission catalog source now lives in backend `config/permission-catalog.json`;
`python scripts/generate-permissions.py` regenerates the runtime catalog. This
does not itself grant permissions to ordinary member roles.

A proposal names type, indicative lower/upper values (or no numeric value for
QUALITATIVE), explanation, methodology/origin, validity period and evidence
snapshot. VALUE/BAND require sufficient evidence at proposal and publication,
and publication refuses stale evidence references. CONVENTION/QUALITATIVE may be
published with insufficient data; the UI still shows that insufficiency.

Publication records the authenticated publisher, decision text, monotonic version
and validity dates. Serialization uses a tenant/definition transaction advisory
lock and unique database constraints, without UPDATE privileges on immutable
tables. The latest publication supersedes previous versions immediately; once it
expires, older versions do not silently become current again. No update/delete
endpoint exists. Database privileges and immutable triggers protect definitions,
policies, observations, snapshots, proposals, publications and agreement context.

v0.1 uses a delegated publisher, not a claim of democratic consensus. The same
person may propose and publish if granted both permissions. Quorum, ballots,
separation-of-duties rules, revocation decisions and governance disputes are the
next increment only when the actual community workflow requires them. Publication
must not be represented as `adminUpdatePrice()`.

This is *ordinary* community governance, never Seven Keys: `publish()`, `policy()`,
`propose()`, `create()` and the market integrity signal/decision endpoints
check `stir.references.*` tenant permissions as usual, but that permission
check alone is not the authority boundary. idax-core's permission service
grants every `stir.*` permission string to a platform superuser
unconditionally (a platform-wide property, not specific to this domain; see
`GOVERNANCE_CAPTURE_THREAT_MODEL.md`), so **platform capability grants must
never be interpreted as community governance authority**. The real boundary
is `ReferenceService.requireCommunityAuthority(CurrentUser user)`, a single
reusable check called as the first statement inside every one of those six
service methods - the authoritative layer, reachable by any future caller,
not only a controller - with `ReferenceController`/`MarketIntegrityController`
calling the same method again at the top of their handlers as
defense-in-depth. Platform administration (creating/enabling a workspace)
must never imply community governance (publishing its references).

## Agreement context and osTRIS boundary

`reference_context_snapshot` is a separate immutable, party-only record captured
in the acceptance transaction. It freezes the definition, community decision,
descriptive snapshot and policy visible at that time, records aggregate consent,
and links to the exact accepted AgreementSnapshot digest. It has a random nonce,
its own RFC 8785 canonical JSON and SHA-256 digest, and explicitly says
`contractual: false` and `CURRENT_AT_ACCEPTANCE_NOT_PROOF_OF_DISPLAY`.

This records what was **current at acceptance**, not a claim that both browsers
displayed or read it. Existing AgreementSnapshot schema/version/hash and
TradeService's contractualMetadataDigest stay unchanged. A later community version
cannot change either historical record. The context digest binds the contractual
digest in one direction; it is not independently anchored in today's osTRIS
authorization. If future requirements demand anchoring both, design a versioned
STIR metadata envelope that commits separate contractual and non-contractual
digests and pass only its digest through the existing opaque metadata field.
That would need explicit compatibility/signing review, not an osTRIS goods/price
protocol extension. No such migration is silently introduced here.

## Manipulation and insufficient evidence

| Case | v0.1 behavior |
|---|---|
| Five agreements between the same two accounts | LOW_DIVERSITY + CONCENTRATED, no amounts |
| Many counterparties trading through one account | CONCENTRATED despite count/diversity |
| Related accounts pretending to be independent | Account concentration can detect repeated accounts, not undisclosed identity links; UI reports this limitation |
| Extreme listing/proposal never accepted | Not an AGREEMENT; excluded with source reason |
| One extreme accepted agreement | SMALL_SAMPLE; no fabricated range |
| Two observations | INSUFFICIENT_DATA; amounts/counts suppressed |
| Old observations | OUTSIDE_WINDOW or STALE, no silent recency assumption |
| Real availability shift | Observed distribution can move at later daily cuts; normative reference does not auto-update |
| Deliberate reference away from observations | Allowed with explicit explanation and publication record |

Scope and comparison attributes preserve room for future community rules about
essential needs. They have no scarcity markup, social-value scoring, urgency
optimization, reputation, personalized recommendation, machine learning or price
blocking behavior. Scarcity and social significance remain distinct questions.

## Validation and operation

Backend: `mvn verify` includes real PostgreSQL Testcontainers, empty Flyway V1–V5,
forced RLS, immutable publication, canonical reconstruction, sample thresholds,
concentration, source distinctions, stale evidence and existing contract tests.
Frontend: `npm test`, `npm run i18n:validate`, `npm run build`.

Local development demonstrations in stir-main:

```text
python scripts/community_references_e2e.py
python scripts/community_references_browser.py
```

They create separate local test tenants/users and real agreements. Never point
these fixture generators at production. The HTTP check covers publisher/member
permissions, tenant A/B, unauthorized third parties, bilateral opt-in, values
below/within/above guidance, new reference versions and unchanged historical
context. The browser scenario creates and publishes a definition/reference,
associates a listing, negotiates below/above the guidance and verifies historical
v1 after v2 publication. Screenshots are local artifacts, not public participant
data. The daily statistical progression is tested with timestamped PostgreSQL
fixtures rather than changing a machine clock or backdating real business rows.

## Deliberately deferred

Verified related-account grouping and independent-person counts; differential
privacy/formal disclosure budgets; consent withdrawal/retention workflow; richer
quantity conversions; multi-community marketplaces per tenant; voting/quorum;
publication revocation/supersession decisions; imports of listing/wanted/seed
evidence; independent context anchoring; and extraction into a shared service
before a second consumer exists. A frontend policy editor (window/minimums/
freshness, inside constitutional floors) shipped with Community Value Governance;
still deferred is any UI for the protected constitutional fields themselves -
those remain reachable only through a signed 7-of-7 Seven Keys amendment.

## Technical observations relevant to a future editorial discussion

These are engineering findings, not edits to a book: accepted, settled and
delivered are different forms of evidence; account diversity is not social
independence; an honest “we do not know” sometimes requires withholding a number;
and preserving an older decision is as important as allowing a new decision.
No files under the private book directory were accessed or modified.
