# WebAuthn / hardware-backed Seven Keys credentials

A Seven Keys seat or the Guardian may be backed by a real WebAuthn/hardware
credential (a security key or a platform authenticator) as an alternative to
the original same-device SOFTWARE_ED25519 test ceremony
(`SEVEN_KEYS_GOVERNANCE.md`). This never replaces 7-of-7, Guardian
separation, or any other constitutional invariant; it changes only how a
credential proves it signed an exact payload. `CredentialEnvelope`
generalizes that proof across both types - a plain Ed25519 signature string
for SOFTWARE_ED25519, or a three-part WebAuthn assertion
(signature/clientDataJson/authenticatorData) for WEBAUTHN. Every new field
this adds is additive: a request that omits `credentialType`/`algorithm`/an
envelope resolves to SOFTWARE_ED25519/Ed25519, byte-for-byte the same
behavior as before this capability existed.

`WebAuthnCrypto` is a minimal, purpose-built registration/assertion
verifier, not a general WebAuthn relying-party library: it parses exactly
the CBOR/binary structures this codebase checks (clientDataJSON,
authenticatorData, a COSE_Key public key) and verifies a signature over
them. Supported algorithms are ES256, RS256 and EdDSA (RFC 8152 §8 -2557
-8/-257). Attestation is deliberately fixed to format `"none"`: no
attestation trust chain, no certificate verification, no metadata service,
no AAGUID allowlist. A clear security benefit was never shown for requiring
attestation here, and requiring it would add a real operational burden
(vetting authenticator models) with no corresponding gain, since STIR's own
constitutional signature verification is what actually protects governance
authority.

Registration (`navigator.credentials.create()`) is a two-step ceremony
owned by `WebAuthnCredentialService`, which never itself grants authority -
a verified credential only becomes constitutionally meaningful once its
resulting public key is submitted through the existing bootstrap/proposal
flow, exactly like a locally-generated Ed25519 key today.
`beginRegistration` issues a single-use, expiring (5-minute), actor/
authority/context-bound challenge; `context` (e.g. `"seat-3"`, `"guardian"`,
`"seat-3-incoming"` during a pending rotation) is purely an anti-mixup label
the caller must echo back unchanged, never itself constitutional authority.
`finishRegistration` verifies the attestation, then persists the WebAuthn
material (credential id, COSE key, initial sign_count, whether user
verification was required) in `stir.constitutional_webauthn_credential`,
separate from `stir.constitutional_seat`'s generic `public_key` column.

Assertion verification (`navigator.credentials.get()`) binds to an exact
governance payload the same way Ed25519 does, but through the WebAuthn
challenge rather than a raw signed message: `challenge =
base64url(SHA-256(domain || 0x00 || the exact domain-separated JCS
payload))` - the same bytes `SevenKeysCrypto.message()` already builds for
Ed25519, just hashed once more before becoming the WebAuthn challenge. A
different proposal, community, tenant, constitution version or seat
produces a different payload, hence a different digest, hence a
`clientDataJSON.challenge` mismatch - a captured assertion cannot be
replayed for anything else. Origin and RP ID are checked against
`stir.webauthn.allowed-origins`/`stir.webauthn.rp-id`
(`STIR_WEBAUTHN_ALLOWED_ORIGINS`/`STIR_WEBAUTHN_RP_ID`, defaulting to
`http://localhost:8089`/`localhost` for local development - a real
deployment must override both to its real public origin/hostname). A
cross-origin assertion is rejected outright.

The authenticator's own `sign_count` provides native clone/replay
detection, independent of and in addition to STIR's own "seat already
signed this proposal" dedup: once a credential's stored counter has ever
been nonzero, every later assertion must report a strictly greater value,
including a drop back to a lower or equal value - the spec's own signal of
a possibly cloned authenticator. This means a literal replay of the exact
same captured assertion for a second submission is rejected as
`400 Invalid constitutional signature` by the sign-counter check, one layer
before STIR's own `409 Seat already signed` dedup ever runs - a stronger
property than Ed25519's plain-signature replay handling, not a weaker one.
Proving the app-level dedup independently therefore needs a genuinely
fresh, validly-signed envelope for an already-signed proposal, not a raw
replay (`VALIDATION_WEBAUTHN_HARDWARE_CUSTODY.md` has the concrete case).
An authenticator whose counter has never left zero (many platform
authenticators with resident keys) is exempt from this check by
construction. User verification (PIN/biometric) is required by default
(`requireUserVerification` defaults to `true` client-side) but is
independently re-checked server-side per credential
(`user_verification_required`, captured at registration) - the server never
trusts the client's own claim alone.

Non-extractable WebCrypto/WebAuthn key material means the private key
cannot be exported off the device holding it - real, but it does not mean
the key can't be misused in place by a compromised client (a malicious
extension, a compromised dependency/OS could still ask the credential to
sign an attacker-chosen challenge while unlocked). WebAuthn is a stronger
guarantee than governance-signer.js's own non-extractable WebCrypto Ed25519
keys (the key material truly never touches this origin's JavaScript at
all, unlike a WebCrypto `CryptoKey` which at least theoretically could be
exposed if the API allowed it), not an absolute one - never phrase either
as "cannot be used by an attacker."

## Migration (V16)

`public_key` (seat/history/authority) and `signature_base64url`
(constitutional_signature) widen from fixed-size `varchar` to `text`: a raw
32-byte Ed25519 key/64-byte signature fit the old sizing, but a COSE EC2/
P-256 key (~120 base64url chars) or an RSA-2048 assertion signature
(~342 chars) do not. `credential_type`/`algorithm` are added to
`constitutional_seat`, `constitutional_credential_history` and
`constitutional_authority` (as `guardian_credential_type`/
`guardian_algorithm`), defaulting to `SOFTWARE_ED25519`/`Ed25519` so every
existing row is unaffected. `constitutional_signature` gains
`client_data_json`/`authenticator_data`, both `NULL` for an Ed25519 row - a
stored WEBAUTHN signature must be independently re-verifiable at activation
time exactly like Ed25519 already is, which needs the full assertion
envelope, not just the raw signature bytes (those alone cannot be
re-verified without the challenge-binding clientDataJSON/authenticatorData
they were computed over).

Two new tables, both RLS-enforced the same way every other `stir.*` table
is: `constitutional_webauthn_credential` (id, tenant, authority, the
seat/guardian's `credential_id`, the WebAuthn `webauthn_credential_id`,
`rp_id`, `sign_count`, `user_verification_required`) and
`constitutional_webauthn_challenge` (registration-ceremony challenges
only - signing/assertion "challenges" need no separate table at all, since
they are deterministic digests of an already-unique, already
domain-separated governance payload, inheriting single-use from the
payload's own uniqueness plus the pre-existing
`constitutional_signature(proposal_id, seat_ordinal)` constraint).
`authority_id` on the credential table is not a foreign key: a credential
is registered and verified before the authority row exists during
bootstrap, exactly like `credentialId`/`controllerId` are already
client-generated UUIDs before bootstrap submits them. No attestation
statement, transport list or AAGUID is stored - none are needed for
verification (attestation is deliberately not required), and storing them
would be gratuitous authenticator fingerprinting.

## Frontend

`webauthn-signer.js` wraps the real `navigator.credentials.create()`/
`get()` calls and shapes their output into the wire payloads the backend
expects, mirroring `governance-signer.js`'s own per-`(authorityId, role)`
IndexedDB convention (`rememberCredential`/`recalledCredential`) so a later
signing action on this same device can find which physical credential
handles a role - a WebAuthn credential can only ever be invoked again from
the device/browser that registered it, WebAuthn's own model, not a STIR
limitation. `challengeFor(messageBytes)` computes
`base64url(SHA-256(messageBytes))`, the same digest construction
`WebAuthnCrypto` verifies server-side.

`governance.jsx`'s `CredentialRegistrar` component lets any seat/Guardian
registration point (bootstrap's `SeatRow`, `APPOINT_GUARDIAN`, and the
`ROTATE_CREDENTIAL`/`REPLACE_CONTROLLER` incoming-credential picker) choose
between the original same-device test ceremony (now explicitly labeled
"TEST CEREMONY / NOT DISTRIBUTED CUSTODY", not distributed custody) and a
real WebAuthn/hardware-backed credential. Registering a credential here
never itself grants authority, mirroring the backend guarantee.

**Known gap, not fixed in this increment:** `CredentialRegistrar` replaced
`ProposeForm`'s old plain "paste a public key" input for the
`ROTATE_CREDENTIAL`/`REPLACE_CONTROLLER` incoming-credential picker with a
generate-or-register-only choice. This means a public key genuinely
obtained on a different device (via the cross-device signing tool's
INVITATION/CONTRIBUTION paste flow, still fully intact for bootstrap's
`SeatRow`) can no longer be typed into that specific rotation form - the
operator must generate/register the incoming credential on the same device
running the rotation proposal UI. This is a real, narrowed capability for
that one form, found while fixing `governance_ui_browser.py`'s regression
(`VALIDATION_WEBAUTHN_HARDWARE_CUSTODY.md`), not something this increment
was scoped to redesign. If genuine cross-device rotation of a seat's
incoming credential becomes a real requirement, `CredentialRegistrar` needs
a third, explicit "I already have a public key" input path, not a silent
workaround.

## Real bug found only by running the real HTTP E2E

`SevenKeysService.verify()` originally read `webauthn.rpId` as a bare field
access from another class. `WebAuthnCredentialService` is `@Transactional`,
so Spring hands callers a CGLIB subclass proxy; CGLIB intercepts *method*
calls and correctly delegates them to the real target (why
`webauthn.allowedOrigins()`, `webauthn.credentialFor()` and the
controller's own `beginRegistration()`/`finishRegistration()` calls all
worked), but a raw *field* read on the proxy reference resolves against the
proxy object's own never-injected field slot, not the target's - silently
returning `null` instead of the `@Value`-injected value. Every WebAuthn
bootstrap/signature verification threw `NullPointerException:
"expectedRpId is null"` as a result. No Postgres/JUnit test caught this,
because `WebAuthnSevenKeysPostgresTest` (and every other `*PostgresTest` in
this codebase) constructs `SevenKeysService`/`WebAuthnCredentialService`
directly via `new`, bypassing Spring's container and therefore its
proxying entirely - only a real, running Spring Boot application driven
over real HTTP exercises the genuine, proxied wiring. Fixed by adding a
package-private `rpId()` accessor method, mirroring the pre-existing
`allowedOrigins()` method. See `VALIDATION_WEBAUTHN_HARDWARE_CUSTODY.md`
for the exact failure and fix.
