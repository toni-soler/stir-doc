# Credential recovery and controller replacement

The Guardian is a separate credential, not an eighth constitutional vote. With
an exact signed payload naming authority, community, seat, credential, reason,
evidence references, expected governance sequence and informative timestamp, it
can perform `EMERGENCY_SUSPEND_CREDENTIAL`. This leaves the controller in place,
invalidates the credential for new signatures, appends a permanent event and
freezes constitutional amendments. A second unilateral suspension fails while
one seat is not ACTIVE. Guardian removal by 5/7 remains available during a
freeze. A removed Guardian immediately loses suspension and recovery authority.

Same-controller rotation is an immutable `ROTATE_CREDENTIAL` proposal. It names
the old and new credentials, new public key, unchanged controller, reason and
continuity evidence references. All six other active seats sign the exact
proposal (6-of-6); the new key signs a separate possession domain; the active
Guardian signs the same proposal in its execution domain. The Guardian cannot
alter the payload or supply a missing seat signature. Activation appends old-key
REVOKED and new-key ACTIVE history, preserves the controller and ends the freeze
only after all seven seats are active. Reusing a credential or another seat's
public key is rejected. The six signatures attest the continuity assertion;
STIR does not claim to have independently verified a real person's identity.

`REPLACE_CONTROLLER` is deliberately **fail closed** in the current runtime.
Its proposal shape distinguishes a changed controller and references a purported
FINAL resolution, but activation returns
`FINAL_RESOLUTION_VERIFICATION_UNAVAILABLE` even with six signatures, Guardian
execution and new-key possession. The inspected osTRIS API has generic
Finding/Appeal/ResolutionBasis persistence and private IdentityContinuity reads,
but no authenticated, tenant-scoped verification endpoint or transferable proof
for a FINAL controller-replacement resolution. Accepting a client-supplied
`finalResolutionId` as proof would make the Guardian/recovery path a superkey.

**SPEC GAP:** define a generic, privacy-preserving final-resolution verification
contract with community sequence, finality/appeal state, resolution digest,
subject, scope and authority. It belongs in osTRIS only if generic across its
consumers; it must not mention goods, market references or price. Add normative
vectors and PostgreSQL/HTTP tests there, then wire a narrow STIR adapter. Until
that exists, permanent incapacity or removal of a controller can lock the seven
keys. No 6-of-7 constitutional fallback is introduced to conceal the gap.

The Guardian cannot approve amendments, publish references, alter policies,
authorize transactions, modify balances, lower quorums, choose a successor or
elect its own replacement by possession of the Guardian credential. A human who
also holds a separately granted STIR publisher role is not cryptographically
prevented from using that ordinary role; role separation and real-world
independence remain community governance duties.
