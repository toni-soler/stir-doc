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

The engineering observation for the private editorial laboratory is that
**valid exchange**, **eligible evidence** and **legitimate rule change** are
three independent judgments. A system can prove which keys signed a proposal
and which facts were excluded, not that a person was morally corrupt or why a
different person did not sign. This document does not edit or publish the book.
