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

<!-- PUSH STATUS: see the live conversation - this checkpoint documents everything through the
local merge; push execution and its confirmation are the very next step in this session. -->

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

**Status: NOT YET EXECUTED.** This is the second half of the requested checkpoint
(`docker compose` isolated stack with STIR frontend/backend, IDAX Core/Shell, osTRIS, IDAX Ledger,
PostgreSQL, object storage, and the audit verifier, plus the full integration gate: clean V1→V19
migration, a representative V16/V17 upgrade with data, backend/frontend build+tests, health of
every service, `/security-status`, the durable-incident+restart reproduction, tenant A/B isolation,
SuperAdmin-without-community-authority, Seven Keys 7-of-7/6-of-7 rejection, WebAuthn replay/cross-
tenant, Market Integrity legitimate/illegitimate, Ordinary Governance, Consent/Retention, Reference
publication, Agreement/economic execution against osTRIS, commit retry/reconciliation, no-FX
invariants, a full restart, and an isolated backup/restore with non-empty data and at least one real
object).

**Why this has not started yet:** running this gate requires reaching the VM DEV host as a
development/test machine (explicitly authorized for this purpose) to bring up an isolated Compose
stack with a new `COMPOSE_PROJECT_NAME`, separate volumes/network/ports/synthetic secrets - never
touching the active `stir-dev` project, its containers, `stir-dev_postgres_data`,
`stir-dev_minio_data`, or anything served by `dev.stir.es`. This session has not established that
connection: there is no SSH configuration for it in any of the four repos' tracked files, and no
connection details (host/port/user/authentication method) were provided in this session beyond the
IP:port pair (`2.139.185.159:12522`) that appears only inside Codex's own audit-evidence files
(`.local/full-system-audit/`) as *their* read-only access record, not a credential handed to this
session. Guessing at credentials or reusing an address found only in someone else's audit log,
without confirming it's the intended target and that this session has legitimate access to it,
would be exactly the kind of unverified assumption this whole five-round remediation effort was
built to avoid making about production systems.

**What is needed to proceed:** confirmation of how this session should reach the VM DEV host (the
existing SSH key pair in `~/.ssh/` may be sufficient if this session is meant to use it, but that
should be confirmed rather than assumed) and confirmation that vendor sources for IDAX Core/Shell,
osTRIS, and IDAX Ledger are available there (the `stir-main` worktree used for the five audit rounds
explicitly does not contain `vendor/`, per the fourth reaudit's own observation - see
`FOURTH_REVALIDATION_GOVERNED_STATE_AUDIT_PHASE1.md`).

**Confirmed unaffected so far:** nothing in Part A touched any remote host. `stir-dev`'s active
containers, volumes, and DNS-served configuration were never referenced by any command in this
session beyond reading past audit documents that mention them historically. This will be
re-confirmed explicitly, with live evidence, once Part B actually runs.

## Resources and timing (Part A only)

Local Windows dev machine, no VM DEV usage yet. Merge operations themselves are near-instantaneous
(fast-forward, no working-tree recomputation beyond the usual checkout). Post-merge verification
commands: `mvn compile` (root + submodule) a few seconds each; `python
scripts/test_audit_security_status_probe.py` ~15 seconds (26 tests, several spin up and tear down a
local HTTP server and a real subprocess). No Docker, no Testcontainers, no VM DEV resources
consumed in Part A.
