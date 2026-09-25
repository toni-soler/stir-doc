# Seven Keys governance UI — change and validation record

Before this increment, Seven Keys / Market Integrity had a complete, tested
backend (`org.stir.reference.SevenKeysService` et al.) but no frontend: the
only way to bootstrap an authority, sign a proposal or suspend a credential
was `curl`/Python (`stir-main/scripts/market_integrity_e2e.py`). This
increment adds the guided ceremony UI described in `SEVEN_KEYS_GOVERNANCE.md`
without changing any backend authorization or cryptographic rule. All three
repositories use local branch `claude/seven-keys-governance-ui-mvp`.

## Exact changed files

**stir-backend**

```text
src/main/java/org/stir/reference/ReferenceController.java   (+GET /community)
src/main/java/org/stir/reference/ReferenceService.java      (+binding())
src/test/java/org/stir/reference/ReferencePostgresTest.java
src/test/java/org/stir/reference/SevenKeysCryptoTest.java   (+shared golden vector)
```

`GET /api/stir/tenants/{tenantId}/references/community` exposes the tenant's
own osTRIS community id (already resolvable indirectly once a reference
definition exists) so the coordinator can start a bootstrap ceremony before
any reference has been created. Same `stir.references.read` permission as
every other read endpoint on this controller.

**stir-frontend**

```text
src/governance-signer.js   (new)
src/governance.jsx         (new)
src/api.js                 (marketGovernanceApi extended; referenceApi.community)
src/extension.jsx          (routes, nav tab)
src/locales.json           (+79 keys x 12 locales = 366 keys each)
tests/governance-signer.test.mjs   (new)
```

**stir-main**

```text
scripts/governance_ui_browser.py   (new)
```

## Design carried over unchanged from the backend

Custody model matches `signer.js` exactly: non-extractable WebCrypto Ed25519
per `(authorityId, role)` in IndexedDB, never sent to STIR. Proposal
signatures always sign the *exact* canonical text the server already
computed (`payloadJson` from the private `/proposals/{id}/signing-payload`
endpoint) - the client never re-derives it. Bootstrap and emergency-suspension
payloads do not exist server-side yet, so the coordinator canonicalizes them
client-side with the `canonicalize` package (RFC 8785 JCS, already a
dependency, cross-verified against the Java `erdtman` implementation via a
shared golden vector in both `SevenKeysCryptoTest` and
`governance-signer.test.mjs`).

Because a real seven-of-seven (plus Guardian) ceremony cannot assume all
eight keyholders share one browser, every signing step supports both a
same-device shortcut and a cross-device relay: export a `{kind:"INVITATION"|
"SIGN", ...}` task as JSON, hand it to the keyholder out of band, they run it
through the generic tool at `/stir/governance/sign`, hand back the resulting
`{kind:"CONTRIBUTION"|"SIGNATURE", ...}` blob. STIR's backend and the
coordinator's browser never see another keyholder's private key.

`REPLACE_CONTROLLER` remains visibly fail-closed in the UI (a warning banner
citing the same `FINAL_RESOLUTION_VERIFICATION_UNAVAILABLE` gap
`CREDENTIAL_RECOVERY.md` documents) - the UI does not hide or work around it.

## Bugs found and fixed during real end-to-end browser testing

The browser E2E (`governance_ui_browser.py`) is what actually caught these;
none were visible from code review or from the Java/HTTP-level tests alone:

1. **Stale authority state after any proposal action.** Signing/activating a
   proposal only refreshed the local proposals list, never the parent
   authority view - the constitution version/digest shown at the top of the
   dashboard stayed frozen at v1 even after a real 7-of-7 amendment had
   already activated to v2 server-side.
2. **Full-page remount on every signature.** The fix for (1) naively called
   the same "loading" transition used for the very first page load on every
   subsequent refresh, unmounting the whole dashboard - and with it every
   `ProposalCard`'s in-progress `payloadJson`/signature state and any open
   `<details>` - after each individual seat signature.
3. **`ProposeForm`'s local constitution snapshot never resynced**, so a
   second `AMEND_CONSTITUTION` proposal in the same session would diff
   against stale pre-amendment values.
4. **The seat-ordinal selector for `ROTATE_CREDENTIAL`/`REPLACE_CONTROLLER`**
   defaulted to seat 1 and never followed the actual (only) suspended seat,
   so the form could visually show the right seat while silently submitting
   the wrong one.
5. **A `<button>` nested inside the same `<label>` as its adjacent textarea**
   polluted that field's accessible name, breaking assistive-technology and
   automated-testing label lookup alike.
6. **An invalid "key must appear in the payload" sanity check** in the
   generic signing tool rejected legitimate Guardian/possession signatures,
   since only bootstrap payloads embed the signer's own public key -
   ordinary proposal payloads never do.
7. Every seat signature triggered a full five-request authority/proposals/
   events/audit/credentials refetch; six signatures in a row produced a long
   trickle of overlapping async updates that kept shifting page layout for
   seconds afterward. Signing/loading a payload now patches only that one
   proposal locally; only activation (which can change seat/constitution
   state) still triggers a full refresh.

All seven are fixed in the committed code, not just worked around in the
test.

## Gate results

| Gate | Result and evidence |
|---|---|
| Backend verify | PASS: `mvn verify`, 118 tests, 0 failures/errors (116 pre-existing + 1 new `binding()` test + 1 shared golden-vector test) |
| Frontend tests | PASS: `npm test`, 34 tests (27 pre-existing + 7 new in `governance-signer.test.mjs`), 0 failures |
| Frontend build | PASS: `npm run build` |
| i18n | PASS: `npm run i18n:validate`, 12 locales, 366 keys each (79 new `gov*` keys) |
| Existing HTTP/browser E2E regression | PASS: `community_references_e2e.py`, `market_integrity_e2e.py`, `community_references_browser.py` all still green after this change |
| **New: governance UI browser E2E** | PASS: `governance_ui_browser.py` - full bootstrap (7 seats + guardian, real WebCrypto Ed25519), 7-of-7 constitutional amendment, guardian emergency suspension, 6-of-6 + guardian + possession same-controller credential rotation via the cross-device relay tool - all through the actually rendered page, no JavaScript page errors |
| Clean Docker | PASS: `docker compose build --no-cache stir stir-ui` |

The browser fixture creates one disposable local development tenant only.
Agent-started `stir`/`stir-ui`/`shell`/`postgres`/`ostris`/`ledger`
containers were stopped after verification.
