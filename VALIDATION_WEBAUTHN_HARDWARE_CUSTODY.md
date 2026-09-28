# WebAuthn Hardware Custody — change and validation record

Full design in `WEBAUTHN_HARDWARE_CUSTODY.md`. Branch
`claude/webauthn-hardware-custody-mvp` across `stir-backend`,
`stir-frontend`, `stir-main`, `stir-doc`. Built on top of the Multi-Source
Value Evidence MVP already on `main` (`stir-backend 53876b3`, `stir-frontend
77c988c`, `stir-main 36c548e`, `stir-doc 5be8f8f`).

This increment has an unusual history worth recording honestly: its
backend/frontend/E2E code was written in an earlier session on a
memory-constrained local machine and never actually run - `mvn verify`
consistently failed there for unrelated reasons (insufficient memory for
Docker+Testcontainers+browser E2E together), so nothing in this MVP had
ever compiled, started, or been exercised against a real backend before
this session moved development to a dedicated VM (`DEV_VM_SETUP.md`) and
ran it for the first time. What follows distinguishes what was verified by
reading the code carefully versus what was verified by actually running
it - the latter is where every real bug in this record was found.

## Exact changed files

**stir-backend** (`8294fb2`, `e127f20`; `93f42a8` merges in an unrelated
Testcontainers/Docker-29 compatibility fix from `main` needed for this
branch's own Postgres-backed tests to run at all on this VM's Docker
Engine 29)

```text
src/main/java/org/stir/reference/CredentialEnvelope.java              (new)
src/main/java/org/stir/reference/WebAuthnCredentialService.java       (new)
src/main/java/org/stir/reference/WebAuthnCrypto.java                  (new)
src/main/resources/db/migration-stir/V16__webauthn_hardware_custody.sql (new)
src/main/java/org/stir/reference/SevenKeysController.java             (+2 webauthn/register endpoints)
src/main/java/org/stir/reference/SevenKeysService.java                (CredentialEnvelope-aware verify(), credentialType/algorithm plumbing)
src/main/resources/application.yml                                    (+stir.webauthn.rp-id/rp-name/allowed-origins)
src/test/java/org/stir/reference/SevenKeysPostgresTest.java           (constructor call site updated only)
src/test/java/org/stir/reference/WebAuthnCredentialServicePostgresTest.java (new)
src/test/java/org/stir/reference/WebAuthnCryptoTest.java              (new, 6 tests - hand-built virtual authenticator)
src/test/java/org/stir/reference/WebAuthnSevenKeysPostgresTest.java   (new)
pom.xml                                                                (+jackson-dataformat-cbor)
```

**stir-frontend** (`9fb2211`)

```text
src/webauthn-signer.js       (new)
src/governance.jsx           (+CredentialRegistrar, CredentialTypeBadge, WebAuthn sign paths in bootstrap/proposal flows)
src/api.js                   (+beginWebauthnRegistration/finishWebauthnRegistration)
src/locales.json             (+8 keys x 12 locales = 560 keys each)
src/style.css                (+.stir-badge-warning)
tests/webauthn-signer.test.mjs (new, 15 tests)
```

**stir-main** (`2c369d5`, `733efcf`, `bbfbba4`; `1d38baf` merges in an
unrelated MinIO quay.io registry-death fix from `main` needed for this
branch's own `docker compose up -d --build` to work at all)

```text
scripts/webauthn_hardware_custody_e2e.py     (new, HTTP E2E)
scripts/webauthn_hardware_custody_browser.py (new, real-browser E2E via a CDP virtual authenticator)
scripts/governance_ui_browser.py             (4 regression fixes - see below)
```

**stir-doc**

```text
WEBAUTHN_HARDWARE_CUSTODY.md            (new - full design)
VALIDATION_WEBAUTHN_HARDWARE_CUSTODY.md (this file)
```

`SEVEN_KEYS_GOVERNANCE.md` and `CREDENTIAL_RECOVERY.md` each got a short
additive paragraph pointing at this document rather than being rewritten -
their existing text about signature verification, rotation and
possession-proof applies unchanged to a WEBAUTHN credential; only *how* a
signature is produced/verified differs, not the governance mechanics they
describe.

No osTRIS/idax-core file was changed. No generator/template affected.

## Design constraints honored explicitly

- **Purely additive wire shape.** Every new field (`credentialType`,
  `algorithm`, `*Envelope`) is optional; every existing call site that
  omits them resolves to `SOFTWARE_ED25519`/`Ed25519`, proven directly by
  `SevenKeysPostgresTest` continuing to pass completely unmodified except
  for its constructor call site (`WebAuthnCredentialServicePostgresTest`
  is a separate new class - the original test file's *assertions* did not
  change at all).
- **No attestation trust chain, by explicit policy.** `WebAuthnCrypto`
  only accepts attestation format `"none"` and never parses `attStmt` -
  proven by `registrationRejectsNonNoneAttestationFormat`.
- **Registration never grants authority by itself** - a credential only
  becomes constitutionally meaningful once its public key is submitted
  through bootstrap/propose/execute, the exact same rule
  `WEBAUTHN_HARDWARE_CUSTODY.md` states, proven implicitly by every E2E
  scenario (registration always precedes and is separate from the
  possession-signature step that actually establishes a seat).
- **Challenge binding prevents cross-context replay.** A captured
  WebAuthn assertion cannot sign a different proposal, community or
  tenant - proven directly (`WebAuthnCryptoTest.assertionRejectsWrongChallenge`)
  and at the HTTP layer against a real backend
  (`webauthn_hardware_custody_e2e.py`'s cross-proposal/cross-community/
  cross-tenant cases).
- **Sign-counter clone/replay detection is real, not decorative.**
  `WebAuthnCryptoTest.assertionRejectsStaleOrClonedSignCounter` proves it
  in isolation; the HTTP E2E proves it fires *before* STIR's own
  already-signed dedup for a literal replay (see "A genuinely surprising,
  correct behavior" below).
- **User verification is independently server-enforced**, never trusted
  from the client alone - `WebAuthnCryptoTest.
  assertionRejectsMissingUserVerificationWhenRequired` proves the server
  rejects an assertion missing UV when the credential's own registration
  required it, regardless of what the client requests at signing time.
- **Guardian's one-suspension-at-a-time invariant applies identically
  regardless of credential type** - the HTTP E2E hit this for real (see
  below) and was fixed by respecting the invariant, not by weakening it.

## Gates

| Gate | Result and evidence |
|---|---|
| Backend verify | PASS: `mvn -s .mvn/public-settings.xml clean verify`, 216 tests (194 pre-existing unaffected + 22 new: 6 `WebAuthnCryptoTest` + `WebAuthnCredentialServicePostgresTest` + `WebAuthnSevenKeysPostgresTest`), 0 failures, migrations V1-V16 applied across every Testcontainers run |
| Frontend tests | PASS: `node --test`, 45 tests (30 pre-existing + 15 `webauthn-signer.test.mjs`), 0 failures |
| Frontend build | PASS: `npm run build` |
| i18n | PASS: `npm run i18n:validate`, 12 locales, 560 keys each (8 new `gov*Webauthn*`/`govCredentialSoftwareTest`/`govTestCeremonyWarning`/`govRegisterHardwareCredential`/`govSignWithWebauthn`/`govNoLocalWebauthnCredential` keys) |
| Existing browser E2E regression | **Initially FAILED, then fixed** - `governance_ui_browser.py` broke in 4 places from `CredentialRegistrar`'s introduction; see "Real bugs found only by running things" |
| **New: HTTP E2E** | PASS: `webauthn_hardware_custody_e2e.py` - mixed-credential bootstrap, valid WebAuthn constitutional signature, replay/cross-proposal/cross-community/cross-tenant rejection, suspended credential cannot sign, same-controller rotation both directions, revoked credentials remain historically verifiable, 6-of-7 still fails, tenant isolation holds |
| **New: real-browser E2E** | PASS: `webauthn_hardware_custody_browser.py` - a seat and the Guardian register and sign with a real (CDP virtual authenticator) WebAuthn credential through the actual rendered UI, across bootstrap and a later constitutional amendment |
| Clean Docker | PASS: `docker compose up -d --build`, every service healthy |

## Real bugs found only by running things (this is the substantive part of this record)

This MVP's code had never actually been executed before this session (see
above). Static reading found nothing wrong - the crypto, the SQL/insert
column-count alignment across the V16 `ALTER TABLE ADD COLUMN` migration,
and the Jackson record-deserialization shape were all independently
verified by careful reading first, and were all in fact correct. Every bug
below was found only by actually running the code, in order:

1. **`docker compose up -d --build stir` was needed after checking this
   branch out on the VM.** The running backend container had been built
   from `main` before this branch's code existed; the first HTTP E2E
   attempt got `404 No static resource ... webauthn/register/begin`
   because the live JVM simply didn't have the new endpoints yet. Not a
   code bug - a deployment-step reminder, but real enough to be worth
   recording since it cost real debugging time before the cause was
   obvious.

2. **`SevenKeysService.verify()`'s `webauthn.rpId` field read returned
   `null`** through the `@Transactional` CGLIB proxy - see
   `WEBAUTHN_HARDWARE_CUSTODY.md`'s "Real bug found only by running the
   real HTTP E2E" for the full explanation. Symptom: every WebAuthn
   bootstrap/signature call failed with `500 Internal Server Error:
   Cannot invoke "String.getBytes(...)" because "expectedRpId" is null`.
   Fixed with a `rpId()` accessor method (`e127f20`).

3. **Six call sites in `webauthn_hardware_custody_e2e.py`** re-fetched an
   already-known, already-immutable proposal's payload via
   `GET /proposals/{id}` (the public, redacted view -
   `SevenKeysService.publicProposal()` strips `payloadJson`) instead of
   using the payload already returned by the proposal's own creation
   response. Symptom: `KeyError: 'payloadJson'`. Fixed by using the
   already-available field directly (`733efcf`).

4. **A genuinely surprising, correct backend behavior, not a bug:**
   replaying the exact same WebAuthn assertion a second time is rejected
   with `400 Invalid constitutional signature` (the sign-counter
   clone/replay check firing inside `verify()`), not the `409 Seat
   already signed` the test originally expected by analogy with Ed25519's
   plain-signature dedup. This is architecturally correct and a stronger
   property, not a weaker one (see `WEBAUTHN_HARDWARE_CUSTODY.md`) - fixed
   the test's expectation and added a second case with a genuinely fresh
   envelope (new sign_count) to still prove the app-level dedup
   independently (`733efcf`).

5. **Test sequencing violated the Guardian's one-suspension-at-a-time
   invariant.** Seat 7 was suspended (to prove "suspended cannot sign"),
   then seat 1's suspension was attempted while seat 7's suspension was
   still unresolved, correctly rejected with `409 One suspension at a
   time`. This is `GOVERNANCE_CAPTURE_THREAT_MODEL.md`'s real invariant
   working as designed, not a bug - fixed by reordering so seat 7's own
   WEBAUTHN->WEBAUTHN rotation (which needed no second suspend call, since
   it was already `EMERGENCY_SUSPENDED`) resolves first, then seat 1's
   SOFTWARE_ED25519->WEBAUTHN suspend+rotate proceeds cleanly (`733efcf`).

6. **`governance_ui_browser.py`, a previously-passing browser E2E, broke
   in four places** the moment `CredentialRegistrar` shipped, discovered
   while validating this MVP's own new browser E2E against the same page:
   - `SeatRow`'s old "Clave publica: `<code>`" summary text was removed
     entirely (now just a credential-type badge + truncated `<code>`, no
     label). Fixed the locator to check for `<code>` instead of the old
     text.
   - `ProposeForm`'s `ROTATE_CREDENTIAL`/`REPLACE_CONTROLLER` new-credential
     picker lost its plain "paste a public key" input, replaced by
     `CredentialRegistrar`'s generate-or-register-only picker. This is a
     **real, unfixed product capability gap**, not just a test problem -
     see `WEBAUTHN_HARDWARE_CUSTODY.md`'s "Known gap" section. Worked
     around in the test by generating the incoming credential locally
     instead of via the cross-device signing tool.
   - Two signature-import `<label>` texts (`Firma del guardián`, `Firma de
     posesion de la nueva clave`) and one suspend-seat `<summary>` text
     each gained an inline `CredentialTypeBadge` or `"(Importar una
     firma)"` suffix, breaking their `exact=True` matches. Relaxed to
     `exact=False`.
   All four fixed in `bbfbba4`; `governance_ui_browser.py` now passes
   again, confirmed by a full clean re-run.

7. **My own browser E2E's seat-loop range was off by one**
   (`range(7)` instead of `range(6)`), so seat 7 got the default Ed25519
   "Generar una clave" click before my WebAuthn-specific code for that
   same form ran - a `Locator.click: Timeout 30000ms exceeded` waiting for
   a WebAuthn radio label that the Ed25519 path had already dismissed by
   replacing the form with its read-only summary. Found and fixed before
   ever landing on `main`, but recorded because it is exactly the class of
   off-by-one this "actually run everything" discipline exists to catch.

## Why a real-browser E2E, when an HTTP E2E already exists

`webauthn_hardware_custody_e2e.py` simulates a WebAuthn ceremony in Python
(real ES256 keys, real CBOR/authenticatorData bytes, real ECDSA
signatures) and proves the backend's verification logic exhaustively, but
it never touches an actual browser, `navigator.credentials`, or
`webauthn-signer.js`'s real code paths - it is a faithful wire-protocol
proof, not a UI/browser-API proof. `webauthn-signer.test.mjs` proves
`webauthn-signer.js`'s own pure helper functions (`challengeFor`,
`decodeBase64url`, IndexedDB remember/recall) in isolation, but never
calls the real `navigator.credentials.create()`/`get()` at all.
`webauthn_hardware_custody_browser.py` is the one script that does: a
Chrome DevTools Protocol virtual authenticator
(`WebAuthn.addVirtualAuthenticator`, `automaticPresenceSimulation: true`,
`isUserVerified: true`) backs real ceremonies driven entirely through
`CredentialRegistrar`'s rendered UI, proving the full stack - Chrome's own
WebAuthn implementation, `webauthn-signer.js`, `governance.jsx`, the HTTP
API, and `WebAuthnCrypto` - genuinely interoperate, and that a registered
credential's `sign_count` keeps working correctly across two *separate*
ceremonies (bootstrap, then a later constitutional amendment), not just
within one.

## Explicitly not attempted

Per `WEBAUTHN_HARDWARE_CUSTODY.md`: attestation trust chain verification,
a FIDO metadata service, an AAGUID allowlist, storing transport/attestation
data. Per the "Known gap" section: a cross-device paste-a-public-key path
for `ROTATE_CREDENTIAL`/`REPLACE_CONTROLLER`'s incoming credential -
`CredentialRegistrar` would need a third explicit input mode to restore
that capability, a real design decision left open rather than improvised
under this increment's own time budget.
