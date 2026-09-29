# Integration checkpoint — Tamper-Evident Governed State Audit Phase 1

## PHASE 1 INTERNAL IMPLEMENTATION GATE: ACCEPTED (2026-09-30)

Codex's final independent verdict, after five rounds of adversarial reaudit, was **`FASE 1
ACEPTABLE`**. This document freezes the exact commits accepted, records their integration into
`main`, and reports what has and has not yet been validated on an isolated VM DEV stack. It does
**not** change `AUD-012` (stays **HIGH, open**) or the pilot-readiness dictamen (stays **NOT PILOT
READY**), and does not authorize Phase 2, an external anchor, or a consumption gate.

Reauditoría evidence, preserved unaltered, PRE-FIX text intact: `REVALIDATION_GOVERNED_STATE_
AUDIT_PHASE1.md`, `SECOND_REVALIDATION_GOVERNED_STATE_AUDIT_PHASE1.md`, `THIRD_REVALIDATION_
GOVERNED_STATE_AUDIT_PHASE1.md`, `FOURTH_REVALIDATION_GOVERNED_STATE_AUDIT_PHASE1.md`,
`FIFTH_REVALIDATION_GOVERNED_STATE_AUDIT_PHASE1.md`. Codex's final `FASE 1 ACEPTABLE` verdict on the
fifth round's remediation (P1-R5-001) was communicated directly rather than as a sixth written
reaudit document; there is no `SIXTH_REVALIDATION_*.md` file, and this checkpoint says so rather
than inventing one.

## Part A — Ancestry verification, integration into `main`, push

### Exact HEADs accepted and merged (this session, 2026-09-30)

| Repo | Pre-integration `main` | Slot-01 branch HEAD accepted | Post-integration `main` (pushed) |
|---|---|---|---|
| `stir-backend` | `ea48c08267170c272bd8f7e5cd00de791f7da28a` | `dd131da...` (after a trivial post-merge whitespace fix, see below) | same as accepted HEAD |
| `stir-main` | `ee4ef11b8e83b8870d35ca8610c81231506aec56` | `d60c9c3681a2b8e3dacc7a09367642391586e3c0` | same |
| `stir-doc` | `da907adb91cff6244002666d710ea5e571bdba43` | `e9c1e2802ea7b71e32eb029c78be9bc248797077` | same, plus this checkpoint + the small stale-claim corrections below |
| `stir-workspace` | `878b59a9ab9dd05078c2c100f4f1474c07dd36dc` | `898b313149e96af2570be1b35496e0fe77161e14` | same, plus a gate-acceptance CLAUDE.md note |

Exact commit lists integrated (fast-forward, no rewrite, no squash - `git log` on each repo's
`main` shows every one of these individually):

**`stir-backend`** (`ea48c08`..`dd131da`):
`ef0d376` feat(audit): tamper-evident governed state audit MVP, Phase 1 →
`f3cfaa5` fix(audit): remediate P1-RA-001..007 from Codex's Phase 1 reaudit →
`381bf14` fix(audit): remediate P1-R2-001/002, ChainVerifier race, alert dedup →
`92ad00d` fix(audit): P1-R3-001 - security_incident is the durable canonical alert →
`dd131da` chore(audit): trim trailing blank line in V18 migration (found only during this
integration's own `git diff --check` against the pre-Phase-1 base; cosmetic, re-verified against a
real migration run before merging - see "New findings" below).

**`stir-main`** (`ee4ef11`..`d60c9c3`):
`d506f82` feat(audit): wire the audit-provision and stir-audit-verifier services →
`00a1938` feat(audit): add independent security-status monitor probe (P1-R3-001) →
`2cb5b4f` fix(audit): P1-R4-001 - make security-status probe fail closed →
`d60c9c3` fix(audit): P1-R5-001 - reject non-string securityState before set lookup.

**`stir-doc`** (`da907ad`..`e9c1e28`, plus this session's follow-up commits):
`096766c` docs: study governed-state tamper-evident audit architecture →
`bb22850` docs(audit): document Phase 1 of the tamper-evident governed state audit MVP →
`864ce06` docs(audit): update Phase 1 checkpoint with P1-RA-001..007 remediation →
`bd3b680` docs(audit): document P1-R2-001/002, ChainVerifier race, alert dedup fixes →
`0e649e9` docs(audit): document P1-R3-001 durable security-status remediation →
`817a9da` docs(audit): document P1-R4-001 fail-closed probe remediation →
`e9c1e28` docs(audit): document P1-R5-001 fail-closed non-string-state fix.

**`stir-workspace`** (`878b59a`..`898b313`):
`898b313` docs: AUD-012 durable invariant note.

### Ancestry verification performed before merging

For every repo: `git merge-base <old-main-HEAD> <slot-01-HEAD>` returned exactly the old `main`
HEAD - confirming a clean, linear, fast-forward-only ancestry with zero divergent history and zero
possibility of a merge conflict. `git worktree list` on every repo confirmed the slot-01 branch and
the active `main` checkout are worktrees of the **same** underlying repository (shared object
database, no cross-clone fetch needed). `git branch -a` from each `main` checkout confirmed the
Codex reaudit branches (`codex/phase1-*-reaudit`, at their own, different commits) exist alongside
`claude/stir-governed-audit-mvp-slot-01` but were never referenced by any merge command - **zero**
Codex-branch commits entered `main` through this integration; only the Claude-authored commits
listed above did. `git status --short` was clean on all four slot-01 worktrees and all four active
`main` checkouts immediately before merging.

### Merge and push

`git merge --ff-only claude/stir-governed-audit-mvp-slot-01` from each repo's `main` checkout - all
four fast-forwarded cleanly, zero conflicts (expected, given the ancestry check above). `git diff
--check <old-main> HEAD` was run on every repo post-merge; `stir-backend` initially reported one
cosmetic trailing-blank-line warning in `V18__tamper_evident_governed_state_audit.sql` (present
since that file was first written, never caught by this session's own per-commit `git diff --check`
runs because those compared each commit against its *immediate* predecessor, not the full range
back to the pre-Phase-1 base). Fixed with a one-line commit on the slot-01 branch (`dd131da`),
re-verified against a real `mvn test -Dtest=VerifierDetectionTest#coverageRegistryClassifiesEveryStirTable`
run (migration still applies cleanly, V1→V19), then re-fast-forwarded into `main`. All four repos'
`main` are now clean on `git diff --check`.

### Push, confirmed

Pushed with explicit user authorization ("Adelante con push."). `git fetch origin main` +
`git rev-parse main` vs `origin/main` confirmed byte-identical on all four repos immediately after
pushing:

| Repo | Pushed `main` HEAD | local == `origin/main` |
|---|---|---|
| `stir-backend` | `dd131da1e85e7d0415c55fad268cc1467ec7d24b` | YES |
| `stir-main` | `d60c9c3681a2b8e3dacc7a09367642391586e3c0` | YES |
| `stir-doc` | `b8568c05c35cad689f52f260ee522328c7fee782` | YES |
| `stir-workspace` | `e61e7fb5dd7e36c91db6e6014096045678394110` | YES |

(`stir-doc`'s own push landed on the second attempt - the first was blocked by Claude Code's own
tool-permission classifier as a precaution on a publish-type action, same as it initially blocked
the very first `stir-backend` push before the user explicitly authorized proceeding; the second
attempt for `stir-doc` succeeded immediately, same command, same content, no workaround used.)

### Post-integration verification (this session)

- `stir-backend/main`: `mvn -q compile` (root) and `mvn -q compile` (`audit-verifier` submodule) -
  both clean, zero errors.
- `stir-main/main`: `python scripts/test_audit_security_status_probe.py` - **26/26 tests PASS**
  against the merged content (byte-identical to what passed repeatedly in the slot-01 worktree
  across rounds 4-5).
- Full `mvn verify`/`mvn test` suites were **not** re-run a sixth time against `main` in this pass -
  the merged content is byte-for-byte identical to what was already verified exhaustively in the
  slot-01 worktree across five remediation rounds (228/228 backend, 41/41 audit-verifier, 26/26
  probe, each re-confirmed at the end of its own round); a fast-forward merge cannot introduce a
  regression a diff didn't already show. The full suites are scheduled to run again as part of
  Part B's isolated-stack gate below, which is a stronger and more relevant check anyway (real
  integration with the rest of the stack, not just this module in isolation).

### Documentation corrections made as part of this integration (not new claims, just accuracy)

Several `stir-doc` files contained language that was accurate *before* this integration
("implemented in an isolated worktree, never merged", "nada de esto se integró en main") and became
stale the moment the merge above happened. Corrected, in this same session, without touching
`AUD-012`'s severity/status or the `NOT PILOT READY` dictamen anywhere:
- `stir-workspace/CLAUDE.md` - corrected the `AUD-012` note's "never merged" claim; added the
  `PHASE 1 INTERNAL IMPLEMENTATION GATE: ACCEPTED` marker.
- `stir-doc/VALIDATION_GOVERNED_STATE_AUDIT_MVP.md` - added the same marker at the top, preserving
  every prior round's own status line and section content unchanged below it (per this document's
  own long-standing "never rewrite prior rounds in place" discipline).
- `stir-doc/PILOT_READINESS.md` - corrected "es una propuesta en worktree, no un control desplegado"
  and "nada de esto se integró en `main` ni se desplegó en DEV" to reflect the actual integration
  state, while explicitly preserving "no un control desplegado [en DEV/PROD]" and "no se ha
  desplegado en el stack DEV activo" as still-true statements.
- `stir-doc/FULL_SYSTEM_AUDIT.md` - same correction to the `AUD-012` entry's "Ni Fase 1... se
  integraron en main... ni se desplegaron en DEV" line.

None of these four files' treatment of `AUD-004`, `AUD-006`, `AUD-008`, `AUD-009`, `AUD-010`, or the
overall `NOT PILOT READY` dictamen was touched - those findings are outside this integration's scope
entirely and were not part of any of the five governed-state-audit reaudit rounds.

## Part B — Isolated full-stack integration test on VM DEV

**Status: STARTED, BLOCKED at gate item 1 by a real HIGH finding (`INT-P1-001`), now remediated in
a narrow follow-up branch pending its own short independent reaudit - see the next section.** VM
DEV connection was confirmed (`ssh -p 12522 stir-admin@2.139.185.159`, user-provided). Before
touching anything: confirmed the active `stir-dev` stack (10 containers, `stir-dev_postgres_data`/
`stir-dev_minio_data`, `stir-dev_data`/`stir-dev_edge` networks) and an unrelated leftover
`stir-audit-restore-20260928-*` project from an earlier Codex session - neither touched.

Isolated workspace: fresh clone of all five repos' just-pushed `main` (verified HEADs matched
exactly) into `/home/stir-admin/stir-audit-verify/stir/`, `stir-main`'s own `scripts/initialize.py`
pinned vendor (IDAX Core/Shell, osTRIS, IDAX Ledger) with its own synthetic secrets, Compose project
`stir-audit-verify` (the committed `compose.yml` has a hardcoded `name: stir-dev` - overridden
explicitly with `-p` on every call; host ports remapped via env vars to avoid colliding with the
active stack).

**Gate item 1 (clean V1→V19 migration) failed:**

```
ERROR: function digest(bytea, unknown) does not exist
Line: 281, V18__tamper_evident_governed_state_audit.sql
Migration of schema "stir" to version "18" failed! Changes successfully rolled back.
```

`select extname, extnamespace::regnamespace from pg_extension where extname='pgcrypto'` →
`pgcrypto | idax_core`. `idax-core-runtime`'s own migrations install `pgcrypto` - into `idax_core` -
before `stir`'s ever run in the real stack; `stir.flyway_schema_history` confirmed a clean rollback
(V17 was the last recorded row, no partial V18 state). Every downstream service (`runtime-
provision`, `audit-provision`, `stir-audit-verifier`, `shell`/`stir`/`ostris`/`ledger`) depends on
this migration step completing and never started. **Gate items 2-19: NOT RUN**, correctly - not
FAIL, simply unreachable. Per the "stop on HIGH" instruction, no fix was attempted in that moment;
the isolated stack was stopped (`stop`, not `down -v` - `stir-audit-verify_postgres_data` was kept
as evidence of the exact failure state) and this was reported for a decision on how to proceed.

## `INT-P1-001` — HIGH / DEPLOYMENT BLOCKER — V18 cannot install alongside idax-core-runtime's pgcrypto

FOUND → FIXED → REVALIDATED BY CLAUDE (this session; pending Codex's own short independent reaudit
before this branch is fast-forwarded into `main` - not yet merged/pushed there).

**Root cause:** PostgreSQL extensions are unique per DATABASE, never per schema. V18's `CREATE
EXTENSION IF NOT EXISTS pgcrypto WITH SCHEMA stir_audit` silently no-ops the moment `pgcrypto`
already exists anywhere in the database - which it always does in the real stack, in `idax_core`.
V18/V19's own hash functions declare `SET search_path = pg_catalog[, stir_audit]`, which never
includes `idax_core`, so the bare `digest(...)` calls never resolved. This was invisible to every
Testcontainers-based test across all five prior reaudit rounds because those always ran STIR's
migrations against an otherwise-bare Postgres container, where V18 itself was always the *first*
thing to ever install `pgcrypto` - the real stack's actual migration ordering (`idax_core` first)
was never exercised until this isolated full-stack gate.

**Determination that V18/V19 remain correctable in place (done BEFORE any code edit, per the
remediation order's own required first step):** checked every known persistent database via SSH -
- Active `stir-dev`: `select version from stir.flyway_schema_history order by installed_rank desc
  limit 1` → **`16`**. V17/V18/V19 never applied.
- The one existing DEV restore snapshot on the VM (`stir-audit-restore-20260928-pg`, a stopped
  container from an earlier Codex session - started briefly, read-only, to check its own history,
  then stopped again in the exact same state it was found in): max version → **`17`**. V18/V19
  never applied there either. Same `pgcrypto`-in-`idax_core` situation.
- The isolated gate's own failed volume: `stir.flyway_schema_history` stops at V17 with a clean
  rollback - by the remediation order's own explicit clarification, this does not count as "V18
  applied" at all.
- No Testcontainers run (ephemeral by construction) counts either.

**Conclusion: no persistent database anywhere has V18 or V19 applied with `success=true`.** Both
remain pre-release migrations, even though already pushed to `main` - correctable directly, with
full Git traceability, per the remediation order's own explicit authorization for exactly this case.

**Fix (in `claude/integration-pgcrypto-remediation`, `stir-backend` commit `bc455fa`):** grepped
every `pgcrypto`/`digest(` usage across the STIR schema - exactly 4 `digest(x, 'sha256')` calls (3
in V18: `genesis_hash`, `row_digest_pg17_jsonb_text_sha256_v1`, `compute_event_hash`; 1 in V19:
`compute_event_hash_v2`) and 1 `CREATE EXTENSION` statement, and nothing else (confirmed no
`gen_random_bytes`/`hmac`/`encrypt`/`decrypt`/`crypt`/`gen_salt`/`pgp_*` anywhere; `gen_random_uuid()`
used elsewhere is itself native `pg_catalog` since PG13, never pgcrypto's). SHA-256 was pgcrypto's
*only* use in this schema. PostgreSQL has shipped `pg_catalog.sha256(bytea) returns bytea` natively
since PG11 - no extension of any kind. Replaced all 4 `digest(x, 'sha256')` calls with `sha256(x)`;
removed the `CREATE EXTENSION` statement entirely. Did **not** move `pgcrypto`, did **not**
`ALTER EXTENSION ... SET SCHEMA`, did **not** grant the auditor any new access to `idax_core`, did
**not** add `idax_core` to any `search_path`, did **not** create a `pg_catalog` wrapper. STIR's
audit code no longer touches, depends on, or needs to know anything about where `idax_core`'s own
extension lives - closing the class of bug, not just this one occurrence of it.

**Cryptographic compatibility, verified before touching any migration file:** a throwaway
`postgres:17-alpine` container confirmed `sha256('hello world'::bytea)` `=`
`digest('hello world'::bytea, 'sha256')` byte-for-byte, for a text value, an empty `bytea`, and
arbitrary binary input (`\xdeadbeef`) - all three `true`. No stored hash's meaning changes, no
`event_format_version` changes, no domain separator changes, no canonicalization/serialization
changes. All 41 pre-existing `audit-verifier` tests (hash vectors via `recomputeEventHash`, tamper
detection, the P1-R3-001 crash-and-restart reproduction, reconciliation) pass unchanged against the
fix - if a single expected hash had changed, any of these would have failed.

**New regression test (the one this incident was missing) - reproduces the real stack's exact
migration ordering, not just "runs against a bare Postgres":**
`migratesCleanlyWhenPgcryptoAlreadyInstalledInAnotherSchemaFirst` creates a schema named `idax_core`
with `pgcrypto` installed into it **before** running STIR's own V1→V19 - exactly the real
precondition - then confirms not just a clean migration but a real, working end-to-end hash-chain
event (a governed mutation through `idax_app`, verified present in `stir_audit.mutation_event`) in
that exact ordering. `migratesCleanlyWithNoPgcryptoExtensionAnywhere` is the inverse control -
STIR's own migrations must never install `pgcrypto` themselves (that would only mask the dependency,
not remove it). `audit-verifier`: **41 → 43 tests, all passing.** `stir-backend mvn verify`:
**228/228, unchanged.**

**Idax-core-first reproduction on the real VM (not just Testcontainers):** copied the two fixed
migration files onto the existing isolated clone at `/home/stir-admin/stir-audit-verify/stir/
stir-backend/`, brought up a **fresh**, differently-named Compose project (`stir-audit-verify-fix2`
- a clean volume/network, per the remediation order's instruction to never reuse the original
failure's evidence volume) with just `postgres`+`core-migrations`+`module-migrations` - the exact
three containers involved in the original failure. Result: `module-migrations` exited `0`;
`stir.flyway_schema_history` shows **V18 and V19 both `success=true`**; `pgcrypto` remains exactly
in `idax_core`, never duplicated or moved; `stir_audit` schema exists correctly. Stopped this
verification stack afterward (`stop`, kept the volume). Re-confirmed `stir-dev` completely
unaffected: same 10 containers, same volumes/networks, `stir.flyway_schema_history` still at `16`.

**Per the remediation order, stopping here - gate items 2-19 were NOT re-attempted in this pass.**
This fix is staged on `claude/integration-pgcrypto-remediation` in `stir-backend` (`bc455fa`) and
documented here in `stir-doc` on the matching branch - **neither pushed to `main` yet**, pending a
short independent reaudit of this narrow fix by Codex. `AUD-012` stays HIGH/open; `NOT PILOT READY`
stays in effect; no Phase 2, external anchor, or consumption gate.

**Once accepted:** fast-forward both branches into `main`, push, bring up a brand-new isolated stack
from that `main`, and resume the full integration gate from item 1 through item 19.

## Resources and timing

**Part A:** near-instantaneous (fast-forward merges); a few seconds each for post-merge `mvn
compile` and the probe's 26-test suite (~15s). No Docker/VM DEV resources consumed.

**Part B (this pass):** VM DEV SSH + Docker throughout. Isolated clone + `initialize.py` vendor pin:
a few minutes. First (failing) migration attempt: under a minute to fail and roll back cleanly.
`INT-P1-001` root-cause investigation (checking `stir-dev`, the restore snapshot, `pg_extension`
locations): a few minutes of read-only SSH queries. Local fix + full local verification (`sha256`
byte-equivalence check, 43 audit-verifier tests, 228 backend tests): a few minutes total. Re-running
the fixed migration on the VM (fresh Compose project, 3 containers): under two minutes end to end.
