# Seven Keys constitutional governance

STIR has exactly seven stable constitutional seat ordinals per tenant/community
authority and a separate Guardian credential. Bootstrap is one-time and requires
proof-of-possession signatures from all seven seat keys and the Guardian over one
JCS payload containing tenant, community, authority, all seat/controller/credential
bindings, Guardian key and initial constitution digest. A deferred PostgreSQL
constraint verifies that all seven seats exist at commit. Distinct credentials
and public keys are required; neither UUIDs nor keys prove seven independent
humans.

The signing bytes are ASCII `STIR:MARKET:CONSTITUTION:V1`, one `0x00` byte, and
RFC 8785/JCS UTF-8 for the immutable proposal payload. Keys and signatures are
raw Ed25519 encoded as unpadded base64url. The payload includes tenantId,
communityId, authorityId/version, proposalId, actionType, beforeDigest,
afterDigest, sorted affectedFields, reason, sorted evidenceRefs, sequence and the
exact after object. `STIR:MARKET:BOOTSTRAP:V1`,
`STIR:MARKET:GUARDIAN:V1` and `STIR:MARKET:POSSESSION:V1` separate bootstrap,
Guardian execution and new-key possession signatures. Signatures cannot be
replayed across actions, proposals, communities or changed payloads. Each seat
counts once; old or suspended credentials do not count.

`AMEND_CONSTITUTION` and normal Guardian appointment need 7-of-7 **active** seats.
There is no 6-of-7 fallback for constitutional changes. An amendment with six
signatures remains visible as a formal, non-activated proposal. Public views
show the action, before/after digests, required count, signing seat ordinals,
status and a redacted hash-linked audit chain. They do not infer why a seat did
not sign. The full signing payload is restricted to delegated publishers; the
public event list omits private evidence references. The audit endpoint
recomputes JCS bytes, SHA-256, predecessor digests and contiguous sequences.

The currently enumerated protected actions are `AMEND_CONSTITUTION`,
`REMOVE_GUARDIAN`, `APPOINT_GUARDIAN`, `ROTATE_CREDENTIAL` and
`REPLACE_CONTROLLER`. The constitution has explicit fields, not a generic
policy DSL. Provenance, immutable history, the threshold of seven, and the
Guardian's inability to govern cannot be switched off through this API.
Ordinary reference publication still uses STIR's delegated publisher workflow;
the Seven Keys do not sign or set marketplace prices.

Guardian emergency removal needs 5 of 7 active seats, without the Guardian's
signature. It can only remove that Guardian and cannot amend the constitution,
change references, choose a successor or rotate a seat. Five is the defensive
threshold because one seat can be suspended by a compromised Guardian and the
other six must retain a removal path. Appointing a new Guardian is separate and
requires all seven active seats plus proof-of-possession of the new key.

Unanimity raises the cost of governance capture but increases denial-of-service
and liveness risk. A missing seat freezes constitutional amendments. There is
deliberately no hidden superadmin, master key, balance adjustment, force
transaction or force reference. Database superusers and a compromised
application process remain outside this cryptographic threat model; deployment
controls and independent audit are required for them.
