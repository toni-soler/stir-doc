# Market and governance capture: threat model

Market capture changes the observed signal: wash trades, A↔B repetition,
circular activity, concentrated counterparties, and related accounts that look
independent. Governance capture changes which signals count or who can decide:
lowering floors, turning off independence checks, rewriting history, bypassing
publication or capturing recovery. The first is partly statistical; the second
requires explicit authority boundaries and permanent audit.

An attacker can submit genuine Agreements at arbitrary negotiated values. STIR
does not invalidate them or osTRIS transactions. Source tags, bilateral consent,
exact unit comparison, account/pair concentration, temporal spread, sensitivity,
private review events and reason-coded exclusions make their influence
inspectable. An accepted outlier is not automatically discarded. A suspected
cluster remains SIGNAL or UNDER_REVIEW until a separate actor reaches FINAL;
DISMISSED remains eligible. Private cluster evidence is not a public KYC feed.

An ordinary publisher may change reference policy only inside code and
constitution bounds. Protected API fields are rejected. A constitutional
proposal retains its exact before/after diff and partial signatures even if it
never activates. Activation rechecks current seat credentials, current
constitution digest, freeze state and action-specific threshold. An admin HTTP
permission is not a constitutional signature. A Guardian can suspend one key,
but cannot fill that seat's vote; 5/7 can remove a captured Guardian.

Residual threats are explicit. A single human can control many accounts or even
multiple constitutional keys unless an external identity and appointment
process prevents it. STIR does not consume private osTRIS RiskSubject links yet,
so account diversity is not independent-person diversity. Case reviewers can
collude; the event chain makes decisions reconstructible but cannot prove their
motives. A database superuser can defeat application RLS and triggers, so the
database and backups need separate operational controls. Unanimity can make
constitutional changes unavailable, especially while controller-replacement
finality lacks a verified contract. These are not solved by an opaque fraud score
or a superkey.

**Custody is device custody, not identity custody.** `governance-signer.js`
generates each seat/Guardian key as a non-extractable WebCrypto `CryptoKey`:
that key's raw bytes genuinely cannot be exported off the device holding it.
It does not follow that the key is unusable by a compromised client - any
code with access to that origin's IndexedDB and the WebCrypto API (a
malicious extension, a compromised dependency, a compromised OS) can still
ask the `CryptoKey` to sign an attacker-chosen message. The guarantee is
"cannot be exfiltrated as bytes," not "cannot be misused in place." A
hardware-backed credential (WebAuthn or similar) is the natural future
evolution if a stronger guarantee is needed; it is not implemented today.

**Same-device bootstrap is a test fixture, not a ceremony.** The bootstrap
wizard's "generate/sign here" shortcuts let one browser hold every seat's key
- exactly what the local dev stack, the HTTP/browser E2E scripts and the
`stir-pruebas` tenant do, deliberately, for fast iteration. That is one
device holding seven-plus-one credentials, not seven independent custodians,
and the UI says so (a same-device-ceremony warning on the bootstrap step).
Treating a same-device bootstrap as a real community's constitutional
ceremony would silently collapse the whole 7-of-7 threshold to "whoever
controls that one device."

**CATASTROPHIC / MULTI-KEY RECOVERY SPEC GAP.** Same-controller rotation
(`ROTATE_CREDENTIAL`) needs the Guardian plus the other six active seats.
Losing two or more seat credentials at once - or one seat credential plus the
Guardian credential - has no recovery path in the current design: there is no
6-of-6 to ask, and no fallback threshold is introduced to work around that
(no master key, no emergency 5-of-7 constitutional path, no SuperAdmin
recovery, no server-generated replacement key, no quorum downgrade). This is
deliberate fail-closed behavior, not an oversight, and it is the same
engineering posture as `REPLACE_CONTROLLER`'s
`FINAL_RESOLUTION_VERIFICATION_UNAVAILABLE` gap in `CREDENTIAL_RECOVERY.md`:
an unsafe shortcut here would make catastrophic loss a backdoor instead of a
hard problem. A normative design for multi-key catastrophic recovery -
almost certainly needing real-world, out-of-band re-attestation of
seat-holder identity before any credential can be reinstated - remains an
open SPEC GAP.

**PLATFORM ADMINISTRATION MUST NEVER BE INTERPRETED AS COMMUNITY GOVERNANCE
AUTHORITY.** idax-core's `PermissionService.hasPermission(user, permission)`
returns `true` unconditionally when `user.isSuperuser()`, for every
permission string, in every tenant - confirmed by decompiling the vendored
`idax-core-0.4.0.jar` at `es.idynamicsax.idax.service.permission.
PermissionService` and empirically (a superuser token with no tenant role
read a tenant's reference definition). This is a platform-wide property of
every `@PreAuthorize("@permissionService.hasPermission('stir.*')")` check in
STIR, not something this or any STIR increment introduced, and it is out of
scope to patch in the vendored dependency. `@PreAuthorize` alone therefore
cannot be the boundary for a community-governed mutation.

**The fix is a single, reusable, service-layer boundary, not a controller
patch.** `ReferenceService.requireCommunityAuthority(CurrentUser user)`
(package-private static, `org.stir.reference`) is the one authoritative
check: it requires a real user (`actor(user)`) and then rejects
`user.isSuperuser()`. It is called as the *first statement* inside every
sensitive mutation in this bounded context -
`ReferenceService.create/policy/propose/publish` and
`MarketIntegrityService.signal/decide` - so the rejection holds regardless of
which controller, or any future non-HTTP caller (a scheduled job, another
service), reaches these methods. `ReferenceController` and
`MarketIntegrityController` call the exact same method again at the top of
the corresponding handler bodies (`create`, `propose`, `publish`, `policy`,
`signal`, `decide`) as defense-in-depth - a fast rejection before any
database work, never a second implementation of the rule. Ordinary reads
(`definitions`, `view`, `history`, `proposals`, `community`,
`observations`, `evidence-manifest`, market integrity `cases`/`history`,
agreement `context`) deliberately do **not** call this check and stay
permission-gated only, per the explicit design goal: protect mutation
authority, not turn all of STIR inaccessible to a platform administrator. A
platform SuperAdmin has no constitutional seat, no Guardian role and no
community role of their own - being SuperAdmin never substitutes for one.
`SevenKeysController` needs no equivalent guard - its protection is the
Ed25519 signature requirement itself, independent of the HTTP permission
layer, so a superuser token still cannot forge a seat's vote.

Regression coverage: `ReferencePostgresTest` calls `ReferenceService`/
`MarketIntegrityService` directly (no controller, no `@PreAuthorize`, no HTTP
layer at all) with a mocked `isSuperuser()=true` `CurrentUser` and proves
every one of the six mutations rejects it, immediately followed by the exact
same call succeeding for a non-superuser actor
(`platformSuperAdminCannotReachAnyReferenceMutationDirectlyThroughTheService`,
`platformSuperAdminCannotReachMarketIntegrityMutationsDirectlyThroughTheService`).
A companion test switches `TenantContext`/`app.tenant_id` mid-transaction to
prove the guard is not a substitute for tenant isolation and grants a tenant
no authority over another tenant's data
(`tenantADoesNotAcquireAuthorityOverTenantBMutationsThroughThisGuard`).
`community_value_governance_e2e.py` proves the same six mutations reject a
real `admin@stir.test` superuser session over real HTTP, that the same
session's genuinely legitimate platform actions (creating a role, provisioning
a user through idax-shell) still succeed, and that plain reads on the same
controllers still succeed for that session too. A residual, deliberate gap
worth naming explicitly: `observations()`/`evidence-manifest()` (private
participant/amount detail, gated on `stir.references.publish`) are reads, not
mutations, so they remain reachable to a superuser through the same
pre-existing idax-core permission bypass this whole finding is about - fixing
that would require patching the vendored permission check itself, which
stays out of scope here (see above); it is not silently left unrecorded. Any
other `stir.*`-gated controller added later inherits the underlying
permission bypass until its own service layer adds the same explicit
`requireCommunityAuthority`-style check; `@PreAuthorize` alone is never
sufficient for a community-governed mutation, in this module or any future
one.

The engineering observation for the private editorial laboratory is that
**valid exchange**, **eligible evidence** and **legitimate rule change** are
three independent judgments. A system can prove which keys signed a proposal
and which facts were excluded, not that a person was morally corrupt or why a
different person did not sign. This document does not edit or publish the book.
