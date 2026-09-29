# Tamper-Evident Governed State Audit MVP — Fase 1: validación y checkpoint

Estado: **Fase 1 remediada por SEGUNDA vez, tras la segunda reauditoría adversarial independiente de
Codex (`SECOND_REVALIDATION_GOVERNED_STATE_AUDIT_PHASE1.md`, dictamen "FASE 1 REQUIERE NUEVA
REMEDIATION"), reentregada para una TERCERA reauditoría. Fase 2 NO iniciada.** `AUD-012` permanece
**HIGH, abierto** — sin cambio de severidad, por instrucción explícita: el objetivo nunca fue cerrar
`AUD-012`. Este documento es el checkpoint solicitado en `CLAUDE_GOVERNED_STATE_AUDIT_ORDERS.md` y
en las dos órdenes de remediación posteriores: no se ha hecho merge ni push a `main` en ningún repo,
no se ha desplegado en la VM DEV activa (`stir-dev`), no se implementó external anchor ni
consumption gate, y no se empezó Fase 2. Todo el trabajo vive en worktrees aislados bajo
`stir/.local/full-system-audit/slot-01/`. La sección "## Remediación de la segunda reauditoría
(P1-R2-001, P1-R2-002, ChainVerifier race, alert dedup)" es la parte nueva de ESTE checkpoint; "##
Remediación de los 7 findings..." es la parte nueva de la entrega ANTERIOR (P1-RA-001...007); el
resto del documento es el checkpoint original de la primera entrega. Las secciones que quedaron
desactualizadas por cada ronda de remediación están marcadas explícitamente como tal, nunca
borradas o reescritas en silencio.

## Objetivo cumplido, formulado con precisión

Una mutación gobernada hecha directamente con la credencial runtime (`idax_app`, heredada por
`idax_backend`) deja una evidencia DB atómica que ese runtime **no puede modificar ni eliminar**
(demostrado con ataques reales contra roles reales, no simulados). Un proceso verificador
**separado**, con credencial propia (`stir_auditor`), detecta:
- los dos ataques directos de `AUD-012` (constitución no firmada, FINAL de Market Integrity
  forjado) — reproducidos de verdad, no supuestos;
- manipulación de la cadena hash (`STORED_HASH_MISMATCH`);
- intentos de `idax_app`/`idax_admin`/`idax_backend` de tocar el propio journal/head/función/
  trigger — todos denegados por PostgreSQL antes de llegar a este código;
- huecos de cobertura (toda tabla `stir.*` real está clasificada o el CI de migración/test falla).

**Lo que NO se afirma, deliberadamente:** ninguna clasificación es `PASS_AUTHORIZED` — ese valor
no existe en el código (`VerificationVerdict.java` sólo tiene `PASS_CRYPTO`, `PASS_STRUCTURE_ONLY`,
`INDETERMINATE`, `NEEDS_BASELINE`, `VIOLATION`, y `PASS_CRYPTO` nunca se devuelve en Fase 1 — ver
SPEC GAP más abajo). Un ataque autoconsistente (SQL fabricando una historia SIGNAL→UNDER_REVIEW→
FINAL con `actor_id` inventados, o una constitución con un `constitutional_proposal`/firmas
igualmente fabricados por el mismo rol) pasaría como `PASS_STRUCTURE_ONLY`, nunca como
autorización real — exactamente el límite que `GOVERNED_STATE_AUDIT_ARCHITECTURE.md` predijo.

## Remediación de los 7 findings de la reauditoría de Codex (segunda entrega)

Evidencia normativa de revisión: `REVALIDATION_GOVERNED_STATE_AUDIT_PHASE1.md` (rama documental
`codex/phase1-independent-reaudit`, fecha 2026-09-29), leído en su totalidad antes de empezar. Cada
finding se documenta abajo como **FOUND → FIXED → REVALIDATED BY CLAUDE** — nunca "revalidado
independientemente"; esa calificación sólo la puede dar la propia reauditoría de Codex sobre esta
entrega. El texto PRE-FIX de Codex se reproduce sin alterar (traducido/resumido del original en
español donde se cita, nunca reescrito para que "encaje" con el fix).

No se cambió la arquitectura general: mismo trigger `SECURITY DEFINER`, mismos roles
`stir_audit_owner`/`stir_auditor`, mismo `audit-verifier` como proyecto Maven independiente sin
Spring, misma separación `mutation_event`/`stream_head`/`coverage_registry`. Todo el trabajo de
remediación vive en `V19__governed_state_audit_phase1_remediation.sql` (aditiva sobre V18, ningún
statement de V18 editado o reordenado) más cambios Java en `audit-verifier` y un nuevo test en el
reactor de `stir-backend`.

### P1-RA-001 — HIGH — el cursor omite commits tardíos permanentemente

**PRE-FIX (Codex):** `AuditSql.fetchEventsAfter()` ordenaba por `(db_time, audit_event_id)` como
watermark; una transacción A que permanece abierta mientras una transacción B de otro stream
confirma antes y es procesada primero deja el cursor avanzado más allá de A — cuando A confirma
después, la consulta incremental no vuelve a mirarla nunca. Reproducido con dos conexiones reales,
`REAL_VERIFIER_CURSOR_LOSS`.

**FIXED:** `AuditSql.fetchEventsAfter`/el watermark fueron eliminados por completo. La nueva
`AuditSql.fetchUnverifiedEvents(ruleVersion, limit)` es un anti-join puro contra
`stir_audit.verification_result` — "todo evento sin fila de veredicto para esta versión de regla",
sin ninguna suposición de orden de commit. `stir_audit.verifier_cursor` sigue existiendo pero pasó
a ser puramente observabilidad/latencia, nunca el mecanismo de completitud.

**REVALIDATED BY CLAUDE:** `VerifierDetectionTest.lateCommittingTransactionAcrossDifferentStreamIsPickedUpOnNextPoll`
reproduce exactamente el escenario de Codex (tenant tardío con transacción abierta, tenant temprano
confirmado y verificado primero) y confirma que el evento tardío aparece en el siguiente poll tras
confirmar. `concurrentInsertsToSameStreamAreStrictlySerializedWithNoGapOrDuplicateSequence` y
`verifierPassRunningConcurrentlyWithWritersMissesNothing` cubren además same-stream y
verifier-concurrente-con-writers reales (hilos JDBC independientes, no simulados). 29/29 tests
`audit-verifier` PASS.

### P1-RA-002 — HIGH — caída tras veredicto pierde el incidente crítico

**PRE-FIX (Codex):** veredicto, incidente y cursor se escribían en autocommits independientes;
`upsertVerificationResult()` seguido de `IncidentReporter.report()` como dos pasos separados. Un
crash entre ambos deja `verification_result=VIOLATION` con cero filas `security_incident` para ese
evento, y el guard `hasVerificationResult()` hace que el reprocesamiento posterior salte ese evento
sin volver a intentar el incidente. `REAL_VERIFIER_INCIDENT_GAP`.

**FIXED:** `AuditSql.recordVerdictAtomically(auditEventId, ruleVersion, verdict, reason, incident,
cursorName)` escribe veredicto + incidente (si aplica) + avance de cursor en **una sola transacción
JDBC explícita** (`autoCommit(false)` + `commit()`/`rollback()` en el mismo bloque try/catch); un
fallo en cualquier punto revierte todo, dejando el evento sin `verification_result` para que
`fetchUnverifiedEvents` lo recoja de nuevo. `security_incident` ganó una columna `dedup_key text
UNIQUE` (V19) para que un mismo incidente reprocessado nunca duplique fila
(`ON CONFLICT (dedup_key) DO NOTHING`). `IncidentReporter.java` (el escritor separado del finding)
se eliminó — su responsabilidad quedó absorbida dentro de la transacción atómica.

**REVALIDATED BY CLAUDE:** `failureMidTransactionLeavesNoPartialVerdictOrIncident` fuerza un fallo a
mitad de la transacción (JSON de evidencia inválido, que sólo falla en el segundo statement,
DESPUÉS de que el upsert de veredicto ya se ejecutó) y confirma que ni el veredicto ni el incidente
quedan persistidos. `reprocessingAfterSimulatedCrashProducesNoDuplicateIncident` confirma que
reprocesar el mismo evento con el mismo `dedup_key` nunca produce una segunda fila.
`verifierLoopRestartProcessesRemainingEventsExactlyOnce` ejercita el punto de entrada real de
producción (`VerifierLoop.runOneIncrementalPassForTest`) con una instancia nueva simulando un
reinicio tras crash.

### P1-RA-003 — HIGH — upgrade sin baseline genera incidentes falsos e ilimitados

**PRE-FIX (Codex):** una fila `reference_definition` insertada antes de que existiera el trigger
(simulando V16/V17→V18) generó `LIVE_ROW_WITH_NO_AUDIT_EVENT` repetidamente en cada poll, sin
distinguir "legítimamente anterior al audit" de "evadió el trigger". `Reconciler` no tenía ningún
concepto de baseline.

**FIXED:** nueva tabla `stir_audit.baseline_import (table_oid, entity_key_canonical, tenant_id,
imported_at, import_batch)`, poblada una única vez por un bloque `DO` de V19 que hace snapshot de
**todas** las claves ya existentes en cada tabla `COVERED` en el momento exacto en que V19 se
aplica — nunca un evento, nunca una entrada en la cadena hash. `Reconciler.reconcileTable` ahora
excluye del listado de "fila viva sin evento" cualquier clave presente en `baseline_import`.

**REVALIDATED BY CLAUDE:** `baselineImportSuppressesLegacyRowsButNotGenuinelyOrphanedOnes` hace un
upgrade real de dos fases contra un Testcontainers Postgres separado — Flyway `.target("17")`,
INSERT directo de una fila legacy real, luego Flyway sin target (V18+V19) — y confirma **cero**
findings de `Reconciler` para esa fila. El mismo test inserta después una fila **genuinamente
huérfana** (trigger deliberadamente bypasseado vía `session_replication_role = replica`, sólo
alcanzable como superusuario, mismo patrón que `CONSENT_RETENTION.md` ya usa para fixtures) y
confirma que **sí** se reporta como `LIVE_ROW_WITH_NO_AUDIT_EVENT` — probando que `baseline_import`
está acotado al momento del baseline, no es una supresión general.

### P1-RA-004 — MEDIUM — `FINAL` sin `UNDER_REVIEW` recibe PASS estructural

**PRE-FIX (Codex):** `IntegrityRule` sólo comprobaba que el primer evento anterior fuera `SIGNAL`;
una secuencia forjada `SIGNAL(seq=1) → FINAL(seq=2)`, saltándose `UNDER_REVIEW` por completo, con
decisor distinto del originador, obtenía `PASS_STRUCTURE_ONLY`.

**FIXED:** `IntegrityRule.evaluate` fue reescrita para validar la **cadena completa** de
transiciones de un caso, no sólo el primer estado: numeración de secuencia contigua 1..N
(`SEQUENCE_GAP_OR_DUPLICATE`), ningún evento después de un estado terminal
(`EVENT_AFTER_TERMINAL_STATE`), primer evento obligatoriamente `SIGNAL`
(`FIRST_EVENT_NOT_SIGNAL`), y cada transición consecutiva debe ser exactamente
`SIGNAL→UNDER_REVIEW` o `UNDER_REVIEW→FINAL|DISMISSED` — cualquier otra combinación,
**incluyendo `SIGNAL→FINAL` directo**, es `INVALID_TRANSITION`. Sigue sin afirmar más que
`PASS_STRUCTURE_ONLY`; la identidad del actor sigue sin prueba criptográfica.

**REVALIDATED BY CLAUDE:** `signalDirectlyToFinalSkippingUnderReviewIsViolation` reproduce **la
reproducción exacta de Codex** (SIGNAL seq=1, FINAL seq=2, decisor≠originador) y confirma
`VIOLATION`/`INVALID_TRANSITION` — antes era `PASS_STRUCTURE_ONLY`. Matriz negativa adicional:
`sequenceGapInCaseHistoryIsViolation`, `firstEventIsUnderReviewWithoutPriorSignalIsViolation`,
`eventAfterTerminalStateIsViolation`, `decisorEqualsOriginatorOnFinalIsViolation`. El flujo legítimo
`legitimateSignalUnderReviewFinalClassifiesStructureOnly` (ya existente) sigue en
`PASS_STRUCTURE_ONLY`, nunca `PASS_AUTHORIZED`.

### P1-RA-005 — MEDIUM — el hash no compromete metadatos del evento

**PRE-FIX (Codex):** `compute_event_hash()` (V18) no incluía `audit_event_id`, `db_time`, `txid`,
`event_format_version`, `request_id`, `correlation_id`, `authorization_id`, `proposal_id`,
`application_name` ni `client_addr`. Editar esos campos como `postgres` (owner) no rompía la
recomputación independiente de la cadena.

**FIXED:** nueva función `stir_audit.compute_event_hash_v2` (dominio de hash separado
`"STIR-AUDIT-EVENT-V2"`, distinto de v1) que compromete los 24 campos, incluyendo todos los que
Codex listó como omitidos. `emit_mutation_event()` fue reemplazada (`CREATE OR REPLACE`, mismo
comportamiento INSERT/UPDATE/DELETE, misma derivación de `entity_key`/digest) para capturar
`audit_event_id`/`db_time`/`txid`/`client_addr` en variables locales ANTES de hashear (así el input
del hash y la fila almacenada están garantizados a coincidir) y usar v2 para todo evento nuevo.
**Los eventos v1 ya escritos NUNCA se reinterpretan** — `AuditSql.recomputeEventHash` despacha por
`event_format_version` a la fórmula v1 u v2 según corresponda a cada fila.

**REVALIDATED BY CLAUDE:** `hashV2DetectsTamperInEachIndependentlyCommittedField` — no un único
campo, sino un bucle sobre `session_role_reported`, `application_name`, `request_id`,
`correlation_id`, `db_time` y `event_format_version`, cada uno tamperado por separado (fila nueva
por campo) — confirma `STORED_HASH_MISMATCH` en los 6 casos. Confirmado además en el `mvn test`
completo que **todo** evento escrito por V19 en adelante lleva `event_format_version =
'STIR_AUDIT_EVENT_V2'`.

### P1-RA-006 — MEDIUM — clasificación de nuevas tablas no es fail-closed tras V18

**PRE-FIX (Codex):** una tabla nueva creada después de V18 con `GRANT INSERT TO idax_app` y sin
fila `coverage_registry` permitía `INSERT`s reales sin ningún evento de auditoría — el bloque `DO`
de clasificación de V18 sólo corre una vez. Además, el propio test de completitud
(`coverageRegistryClassifiesEveryStirTable`) vivía únicamente en el proyecto Maven independiente
`audit-verifier`, nunca en el reactor de `stir-backend` que ejecuta `mvn verify` — un `mvn verify`
normal nunca lo corre. Codex también señaló que la clasificación `NO_PRIVILEGE_CATALOG` sólo
comprobaba `INSERT`, nunca `UPDATE`/`DELETE`.

**FIXED — dos partes:**
1. Nuevo test **en el reactor de `stir-backend`**:
   `src/test/java/org/stir/audit/StirAuditCoveragePostgresTest.java` (paquete `org.stir.audit`,
   nuevo, plain JUnit5 + Testcontainers, mismo patrón que
   `StirAdminAuthorityPostgresTest` ya existente) — así un `mvn verify` normal en `stir-backend`
   solo sí ejecuta esta comprobación, sin depender del proyecto separado.
2. V19 añade un bloque `DO $do_coverage_privilege_reverify$` que re-verifica, en cada migración,
   que ninguna fila `NO_PRIVILEGE_CATALOG` tenga hoy `INSERT`, `UPDATE` **o** `DELETE` concedido a
   `idax_app` — antes sólo se comprobaba `INSERT` en V18. Falla cerrado (`RAISE EXCEPTION`) si la
   invariante se rompe.

**REVALIDATED BY CLAUDE:** `StirAuditCoveragePostgresTest` — 3/3 PASS:
`everyRealStirTableHasACoverageRegistryRow` (completitud, réplica del test de `audit-verifier` pero
en este reactor), `noPrivilegeCatalogTablesHaveNoInsertUpdateOrDeleteGrantedToIdaxApp` (los 3
verbos, no sólo INSERT — confirmado que `category`/`resource_kind`, las dos únicas filas
`NO_PRIVILEGE_CATALOG` reales, no tienen ninguno de los tres hoy), y
`syntheticUngovernedTableWithRuntimeGrantsTripsTheCoverageGate` — crea una tabla sintética real con
`GRANT INSERT, UPDATE, DELETE TO idax_app` y **sin** fila `coverage_registry`, y confirma que la
consulta de completitud la detecta (no vacuamente, sobre un esquema ya completo, sino sobre una
tabla nueva de verdad). El bloque `DO` de V19 se ejecutó limpio contra el esquema actual (`category`
y `resource_kind` re-verificados en vivo, cero privilegios en los 3 verbos).

### P1-RA-007 — MEDIUM — el verificador no revalida una cadena histórica alterada

**PRE-FIX (Codex):** se insertó y actualizó una fila (`participant_independence_projection`,
INSERT+UPDATE), el cursor avanzó más allá de ambos eventos, y sólo como `postgres` se borró el
evento INSERT más antiguo, dejando el UPDATE y la fila. Tras varios segundos con el proceso activo,
el recuento de incidentes `CHAIN_%` siguió 0→0 — `Reconciler` sólo compara existencia de clave viva
contra el último evento, y nada invocaba `ChainVerifier.verifyStreamFrom()` periódicamente.

**FIXED — dos iteraciones, la primera insuficiente:** el primer borrador añadió
`stir_audit.chain_reconciliation_checkpoint` (tabla, V19) y un paso periódico en `VerifierLoop`
(cada `PERIODIC_RECONCILE_EVERY_N_CYCLES=10` ciclos) que **resumía** desde
`checkpoint.lastVerifiedSequence()+1`. Al escribir el test de regresión para este finding se
detectó, antes de que Codex tuviera que encontrarlo, que resumir desde el checkpoint sólo vuelve a
leer en fresco el evento-frontera del propio checkpoint — un tamper a un evento **interior** ya
superado por el checkpoint nunca vuelve a examinarse en ningún pase posterior, reabriendo
silenciosamente el mismo hueco que este finding existe para cerrar. **Corregido de verdad**:
`VerifierLoop.runPeriodicChainReconciliation` ahora llama siempre a
`chain.reconcileStreamFrom(tenant, domain, 1)` — recamina el stream completo desde génesis en cada
pase periódico, nunca desde el checkpoint. `chain_reconciliation_checkpoint` pasó a ser pura
observabilidad ("hasta qué secuencia confirmó limpio el último recorrido completo"), nunca un punto
de reanudación que se salte eslabones históricos. Coste: O(longitud de la cadena) por pase
periódico — documentado explícitamente como trade-off de Fase 1, no oculto.

**REVALIDATED BY CLAUDE:** cuatro tests, cada uno sembrando un stream de 3 eventos, corriendo un
primer pase limpio que avanza el checkpoint a 3, y **entonces** tamperando:
- `periodicReconciliationCatchesEditedHashOnInteriorEventAfterCheckpointAdvancedPastIt` — edita
  `current_hash` de la secuencia 1 (la más antigua) → `STORED_HASH_MISMATCH`.
- `periodicReconciliationCatchesDeletedInteriorLinkAfterCheckpointAdvancedPastIt` — borra la
  secuencia 2 (interior) → `SEQUENCE_GAP`.
- `periodicReconciliationCatchesDeletedGenesisEventAfterCheckpointAdvancedPastIt` — reproduce **la
  reproducción exacta de Codex** (borra el evento más antiguo, deja sólo el/los posteriores) →
  `MISSING_PREDECESSOR`.
- `periodicReconciliationCatchesIncorrectStreamHeadAfterCheckpointAdvancedPastIt` — tampera
  `stream_head.head_hash` directamente → `STREAM_HEAD_MISMATCH`.

Los cuatro confirman `checkpoint.lastVerifiedSequence()` ya en 3 (superado) **antes** de tamperar,
para ejercitar realmente el hueco que Codex encontró, no una variante más fácil.

### Concurrencia (nota aparte de Codex, no numerada P1-RA-00X)

Codex documentó explícitamente un **gap de concurrencia no validado** en `Reconciler`
(`liveEntityKeys`/`entityKeysExpectedLive` en transacciones autocommit separadas) sin lograr forzar
una intercalación determinista. **FIXED:** `AuditSql.readConsistentTableSnapshot` envuelve las tres
lecturas (`live`, `expectedLive`, `baselineImported`) en una única transacción
`SET TRANSACTION ISOLATION LEVEL REPEATABLE READ, READ ONLY` — el snapshot MVCC de PostgreSQL, fijo
en el primer statement de esa transacción, garantiza que las tres ven el mismo instante, sin ningún
lock de aplicación. **REVALIDATED BY CLAUDE:**
`concurrentInsertsToSameStreamAreStrictlySerializedWithNoGapOrDuplicateSequence` (dos escritores
JDBC reales, mismo stream, hilos concurrentes genuinos vía `ExecutorService`) y
`verifierPassRunningConcurrentlyWithWritersMissesNothing` (10 escritores concurrentes + polling del
verificador corriendo en paralelo, confirmando cero eventos perdidos y cero duplicados en el
conjunto de veredictos).

## Remediación de la segunda reauditoría (P1-R2-001, P1-R2-002, ChainVerifier race, alert dedup)

Evidencia normativa: `SECOND_REVALIDATION_GOVERNED_STATE_AUDIT_PHASE1.md` (rama documental
`codex/phase1-second-reaudit`, fecha 2026-09-29), leído en su totalidad. Esta iteración fue
deliberadamente estrecha: no se rediseñó Fase 1, no se tocaron los fixes ya revalidados de
P1-RA-001/002/005/006/007 salvo donde se indica explícitamente abajo (P1-RA-006 recibió una mejora
adicional, P1-RA-007's checkpoint semantics no cambiaron). Cada punto se documenta como
**FOUND → FIXED → REVALIDATED BY CLAUDE**.

### P1-R2-001 — HIGH — falso CRITICAL en Market Integrity legítimo

**PRE-FIX (Codex):** en `IntegrityRule.java:88-94`, `thisStatus` se leía como el ÚLTIMO estado de
TODA la historia del caso (`history.get(history.size()-1).status()`), mientras que `decisorId` se
leía del actor del evento ACTUALMENTE VERIFICADO (`eventRowId`). Para un caso legítimo
`SIGNAL(originador)→UNDER_REVIEW(decisor)→FINAL(decisor)`, verificar el evento SIGNAL leía
`thisStatus="FINAL"` (el estado final del caso) pero `decisorId`=actor del SIGNAL=el originador →
`VIOLATION/DECISOR_EQUALS_ORIGINATOR` falso sobre el propio SIGNAL. El test de la primera
remediación sólo evaluaba el último evento de forma aislada, nunca el backlog completo, por lo que
nunca ejercitó este caso.

**FIXED:** `IntegrityRule.evaluate` fue reescrita para evaluar, para cada evento, el PREFIJO
histórico que termina EXACTAMENTE en la secuencia de ESE evento (`sequence <= thisSequence`), nunca
la historia completa del caso. `thisStatus`/`thisActorId` se leen directamente de la fila del propio
evento (una única consulta), nunca derivados de "el último elemento de alguna lista". El check
`DECISOR_EQUALS_ORIGINATOR` sólo se aplica cuando el estado del PROPIO evento es FINAL/DISMISSED,
usando el actor del PROPIO evento — nunca el actor de una fila distinta.

**REVALIDATED BY CLAUDE:** `fullBacklogOfLegitimateSignalUnderReviewFinalProducesZeroIncidents`
reproduce exactamente el caso obligatorio de Codex (SIGNAL(A)→UNDER_REVIEW(B)→FINAL(B), A≠B) y lo
procesa a través del pipeline de producción real (`VerifierLoop.runOneIncrementalPassForTest`, no
una llamada directa a la regla sobre un evento elegido a mano) — confirma
`SIGNAL:PASS_STRUCTURE_ONLY`, `UNDER_REVIEW:PASS_STRUCTURE_ONLY`, `FINAL:PASS_STRUCTURE_ONLY` y
**cero** `security_incident`. Matriz negativa ampliada:
`reorderedUnderReviewAfterFinalIsViolation`, `secondTerminalStateAfterFirstIsViolation`,
`eventAfterDismissedIsViolation`, `decisorEqualsOriginatorOnDismissedIsViolation` (además de las ya
existentes de la primera remediación: FINAL sin historia, SIGNAL→FINAL directo, UNDER_REVIEW sin
SIGNAL, gap de secuencia, evento tras FINAL, decisor=originador tras FINAL). Ninguna produce
`PASS_AUTHORIZED`.

### P1-R2-002 — HIGH en garantía de upgrade — baseline no debe prometer origen legítimo

**PRE-FIX (Codex):** V19 importaba TODAS las claves vivas de una tabla COVERED al momento de aplicar
V19, sin cotejar si V18 ya llevaba tiempo activo ni si el trigger había tenido oportunidad de ver
esa fila. Una fila insertada DESPUÉS de V18 (trigger ya activo) pero con el trigger desactivado
manualmente por `postgres` antes de que V19 corriera fue importada como baseline igual que datos
legado genuinos — indistinguible. Codex también advirtió que el propio owner puede manipular
`baseline_import`, por lo que la documentación nunca debe presentar el baseline como prueba
inequívoca de origen legítimo.

**FIXED — se reescribieron V18 y V19 directamente (no publicadas/desplegadas nunca, ver nota más
abajo), no una V20 nueva:**
1. **Semántica corregida explícitamente**, en código y documentación: una fila en `baseline_import`
   significa *ÚNICAMENTE* que ese (tabla, clave) estaba presente en el momento en que la ceremonia
   de activación la capturó — nunca prueba de origen legítimo. Formulación exacta, ahora en los
   comentarios de `Reconciler.java`, `AuditSql.java` y V18 mismo:
   > Baseline records pre-audit state under an explicit trusted upgrade ceremony; it does not prove
   > the historical authorization or origin of those rows.

   Equivalente explícito: `LEGACY_UNVERIFIED`. Nunca `VALID`/`AUTHORIZED`/`AUDITED`.
2. **`baseline_import` y su población se movieron de V19 a V18**, colocadas DENTRO del mismo bucle
   que crea el trigger de cada tabla `COVERED`. Antes de leer una sola fila de esa tabla, el bucle
   ejecuta `LOCK TABLE stir.<tabla> IN ACCESS EXCLUSIVE MODE` — PostgreSQL retiene ese lock hasta que
   la transacción completa de V18 confirma o revierte, así que desde el instante en que el lock se
   adquiere, ningún `idax_app` puede `INSERT`/`UPDATE`/`DELETE` esa tabla hasta que V18 termine por
   completo. Esto cierra técnicamente el hueco "quiesce application writers → capture baseline →
   activate trigger" para el modelo de amenaza runtime ordinario, sin depender de un procedimiento
   operativo manual.
3. **Nueva tabla `stir_audit.activation_marker`** (`activation_batch` PK, `activated_at`,
   `migration_version`), un registro durable y consultable del instante exacto en que "audit
   protection active" se volvió cierto — el marcador de frontera que pedía el diagrama de la
   ceremonia. `baseline_import.import_batch` tiene ahora un FK real a esta tabla.
4. **Explícitamente fuera de alcance, por instrucción directa**: un `postgres`/DB owner/host root que
   desactiva el trigger y escribe ANTES de que esta ceremonia corra sigue siendo indistinguible de
   legado genuino — ese actor está fuera del trust boundary de Fase 1 y esta remediación no finge lo
   contrario en ningún comentario o documento.

**Nota sobre V18/V19 no publicadas:** ninguna de las dos había sido integrada/desplegada en ningún
entorno antes de este commit, así que se corrigieron directamente en vez de acumular semántica
correcta sobre una V20. `git log -p` sobre `V18__tamper_evident_governed_state_audit.sql` y
`V19__governed_state_audit_phase1_remediation.sql` a través de los commits de esta sesión es la
comparación before/after exacta que pide la orden de remediación — nunca se reescribió en silencio
el hallazgo PRE-FIX de Codex en ninguno de los dos documentos de reauditoría.

**REVALIDATED BY CLAUDE:** `baselineImportSuppressesLegacyRowsButNotGenuinelyOrphanedOnes` (ya
existente, sigue pasando sin cambios contra la nueva estructura V18/V19 - upgrade real V17→V19) más
`v16DataMigratesThroughActivationAsLegacyUnverifiedWithoutFalseCritical` (nuevo, upgrade real
V16→V19, sin trigger en absoluto en V16 así que no hace falta `session_replication_role`). Ambos
confirman: datos legado → `LEGACY_UNVERIFIED`, cero CRITICAL; una fila insertada DESPUÉS de la
ceremonia con el trigger deliberadamente evadido (sólo alcanzable como superusuario) → sigue
alertando como `LIVE_ROW_WITH_NO_AUDIT_EVENT`. Ningún baseline se convierte jamás en
`PASS_AUTHORIZED`. `mvn verify` completo del backend confirma que las migraciones V1→V19 siguen
aplicando limpio de principio a fin con la nueva estructura.

### ChainVerifier — snapshot consistente (carrera pendiente, no numerada P1-RA/P1-R2)

**PRE-FIX (Codex):** `ChainVerifier.reconcileStreamFrom()` leía `stream_head` y `mutation_event` en
dos llamadas autocommit SEPARADAS (`ChainVerifier.java:74,78` en el código PRE-FIX). Un commit real
aterrizando entre ambas lecturas podía producir una vista transitoriamente inconsistente (una head
ya avanzada que la lectura de eventos aún no reflejaba, o viceversa), apareciendo como un falso
`EXPECTED_EVENT_MISSING`/`STREAM_HEAD_MISMATCH`. 400 escritores concurrentes no lograron reproducir
la intercalación exacta, pero Codex documentó el hueco como real y no cerrado por prueba.

**FIXED:** nuevo par `AuditSql.beginRepeatableReadSnapshot()`/`endRepeatableReadSnapshot()` — abre
una transacción `REPEATABLE READ, READ ONLY` que permanece abierta a través de MÚLTIPLES llamadas
posteriores (a diferencia de `readConsistentTableSnapshot`, que abre/lee/cierra en una sola llamada).
`ChainVerifier.reconcileStreamFrom` ahora envuelve la lectura de `stream_head` Y la lectura de
`mutation_event` dentro de ESE mismo snapshot compartido — el snapshot MVCC de PostgreSQL, fijado en
el primer statement de la transacción, garantiza que ambas lecturas ven exactamente el mismo
instante. Ningún lock de aplicación. Un commit que llega después del snapshot es simplemente
invisible para ese pase completo (diferido correctamente al siguiente ciclo), nunca parcialmente
visible a una lectura y no a la otra.

**REVALIDATED BY CLAUDE — before/after propio:** exponer `begin`/`end` explícitos (en vez de un
único método caja-negra) permitió escribir una prueba determinista real, sin necesitar matar hilos
ni introducir sleeps: `chainVerifierSnapshotIsNotRacedByWriterCommittingDuringSameStreamReconciliation`
replica manualmente los dos pasos de `reconcileStreamFrom` (lectura de head, luego de eventos) sobre
el snapshot compartido, con un escritor REAL confirmando un evento nuevo al MISMO stream en una
conexión completamente separada, estrictamente entre ambas lecturas — confirma que la lectura de
eventos NO ve el commit del escritor (el snapshot se sostiene), y que el evento diferido se
reconcilia limpio, sin incidente falso, en el siguiente pase fresco.
`chainVerifierSnapshotIsNotRacedByWriterCommittingToDifferentTenantDuringReconciliation` repite lo
mismo con un escritor a un tenant/stream DISTINTO durante la ventana ("otro tenant" per la orden),
confirmando que ambos streams reconcilian limpio después. **Antes** de este fix, replicar el mismo
patrón de prueba (dos lecturas autocommit separadas con un commit real inyectado entre ellas) sí
produce la vista inconsistente que Codex predijo — verificado manualmente durante el desarrollo de
este fix, no incluido como test permanente para no dejar un test que documente un bug ya cerrado
como si aún aplicara al código actual.

### Deduplicación de la alerta operativa (item 4, segunda reauditoría)

**PRE-FIX (Codex):** `security_incident` deduplicaba correctamente a nivel de fila
(`ON CONFLICT (dedup_key) DO NOTHING`), pero el auditor imprimía `level=CRITICAL` incondicionalmente
en cada ciclo que redescubría la MISMA condición persistente (un hallazgo de `Reconciler`, o una
ruptura de cadena ya reportada) — deduplicación de fila en DB ≠ deduplicación de la alerta externa.

**FIXED:** `AuditSql.insertSecurityIncidentTx` ahora devuelve `true` sólo cuando `executeUpdate()`
realmente insertó una fila (0 = deduplicado, 1 = nuevo — sin necesitar `RETURNING`).
`recordVerdictAtomically`/`recordChainIncidentAtomically`/`recordIncidentOnly` propagan ese booleano.
`VerifierLoop` ahora sólo imprime `level=CRITICAL` cuando el booleano es `true`, en los tres puntos
de emisión (veredicto individual, hallazgo de `Reconciler`, ruptura de cadena periódica).
`HealthServer` ganó `openIncidentCount` (actualizado cada ciclo desde `AuditSql.countIncidents()`,
expuesto también en `/health`), separado deliberadamente del log de CRITICAL: un incidente
persistente sigue apareciendo como abierto en `/health` en cada ciclo, sin que eso dispare una nueva
notificación externa.

**REVALIDATED BY CLAUDE:** `persistentIncidentEmitsCriticalOnceAcrossMultipleCyclesButStaysOpenInHealth`
siembra una fila genuinamente huérfana, corre 4 ciclos reales de `VerifierLoop.runOneIncrementalPassForTest`
capturando `System.out`, y confirma exactamente **una** línea `CRITICAL` para ese incidente a través
de los 4 ciclos, mientras `health.openIncidentCount()` permanece ≥1 durante todo el tiempo.

### P1-RA-002 — frontera de crash (item 5, refuerzo, sin rediseño)

Sin cambios a la transacción SQL existente (correcta según Codex). Se añadió el complemento directo
de `failureMidTransactionLeavesNoPartialVerdictOrIncident` (ya existente, prueba "durante la
transacción"): `successfulTransactionLeavesBothVerdictAndIncidentPersistedTogether` prueba el otro
lado de la garantía de atomicidad — tras un `commit` real y exitoso, tanto el veredicto COMO el
incidente están presentes juntos, nunca uno sin el otro. Together, ambas pruebas cubren
"antes/durante/después" sin necesitar matar la JVM entre cada instrucción JDBC, ya que PostgreSQL
mismo garantiza la atomicidad de la transacción.

### P1-RA-006 — evidencia adicional verbo-por-verbo (item 6)

`ungovernedTableWithOnlyOneOfInsertUpdateOrDeleteEachTripsTheGateIndependently` (nuevo, en
`StirAuditCoveragePostgresTest`, reactor `stir-backend`) reproduce exactamente el patrón de Codex:
tres tablas temporales `stir.audit_probe_{insert,update,delete}`, cada una con SÓLO el grant
indicado y sin fila de registry — confirma que la consulta de cobertura atrapa las tres variantes de
forma independiente, no sólo una tabla con los tres grants combinados (que la prueba sintética
existente ya cubría por otro mecanismo). Sin migración V20 de prueba; fixture dinámico dentro del
test, tal como pidió la orden.

### Regresiones obligatorias (item 7) — sin cambios de comportamiento, re-confirmadas

`idaxAppAndIdaxAdminCannotTouchAuditJournalOrHead`, `sessionReplicationRoleCannotBeSetByRuntimeRoles`,
`rolledBackMutationLeavesNoEventAndNoHeadAdvance`, las pruebas de P1-RA-001 (anti-join),
P1-RA-002 (atomicidad), P1-RA-005 (hash v2 independiente), P1-RA-007 (tamper histórico),
`coverageRegistryClassifiesEveryStirTable`/`everyRealStirTableHasACoverageRegistryRow`, y
`tenantAMutationNeverTouchesTenantBStream` - las 39 pruebas de `audit-verifier` (12 originales + 17
de la primera remediación + 10 de esta segunda ronda) y las 3+1 de `StirAuditCoveragePostgresTest`
siguen en verde sin haber tocado su lógica salvo donde se documenta explícitamente arriba.

### Totales confirmados de esta segunda ronda (re-ejecutados en esta sesión, no reportados de memoria)

- `mvn test` en `audit-verifier`: **39/39 tests, 0 fallos** (era 29/29 tras la primera remediación).
- `mvn clean verify` en `stir-backend` (reactor completo): **228/228 tests, 0 fallos** (era 227/227;
  el único nuevo es `ungovernedTableWithOnlyOneOfInsertUpdateOrDeleteEachTripsTheGateIndependently`
  en `StirAuditCoveragePostgresTest`, ahora 4/4). Incluye la migración V1→V19 completa con la nueva
  estructura de activation ceremony aplicando limpio en cada uno de los ~30 módulos de test que
  migran su propia base Testcontainers.
- `docker compose -p stir-audit-remediation-slot01-r2 config --quiet` y
  `... build audit-provision stir-audit-verifier`: **PASS**, re-verificado en esta sesión contra el
  código de esta segunda remediación (proyecto Compose aislado, nombre distinto de la primera
  verificación y de `stir-dev`, ningún contenedor arrancado).
- `git diff --check`: limpio.

## Worktrees y baseline

| Repo | Worktree | Rama | Base | HEAD tras Fase 1 |
|---|---|---|---|---|
| `stir-backend` | `slot-01/stir-backend` | `claude/stir-governed-audit-mvp-slot-01` | `codex/audit-high-gate@ea48c08` (V17 + fix AUD-010 ya integrados) | ver `git log` en el worktree |
| `stir-main` | `slot-01/stir-main` | `claude/stir-governed-audit-mvp-slot-01` | `codex/audit-high-gate-e2e@ee4ef11` | ver `git log` en el worktree |
| `stir-doc` | `slot-01/stir-doc` | `claude/stir-governed-audit-mvp-slot-01` | `codex/governed-state-audit-architecture@096766c` | ver `git log` en el worktree |

Ningún checkout activo (`/c/Users/tonis/source/repos/stir/*`, la VM DEV en `/srv/stir/stir/*`, ni
los worktrees `codex/*` preexistentes) fue tocado. `git status` en cada uno de esos tres sigue
limpio y en su rama original — verificado antes de crear los worktrees nuevos.

## Migración (`stir-backend`)

`src/main/resources/db/migration-stir/V18__tamper_evident_governed_state_audit.sql`, aditiva sobre
V17. Contenido:

1. Roles `stir_audit_owner` (NOLOGIN) y `stir_auditor` (LOGIN, password fijada fuera de la
   migración por `provision-audit.sh`, igual que `idax_backend`).
2. Esquema `stir_audit`, `REVOKE ALL ... FROM PUBLIC`, `GRANT USAGE` sólo a `stir_auditor`.
3. Extensión `pgcrypto` instalada en el esquema `stir_audit` (para `digest()`).
4. Tablas: `coverage_registry`, `stream_head`, `mutation_event`, `verifier_cursor`,
   `verification_result`, `security_incident`, `anchor_outbox` (**interfaz Fase 2, sin
   implementación — todo status queda `NOT_IMPLEMENTED_PHASE1`, ningún código Fase 1 la escribe**).
5. Funciones puras de canonicalización/hash (`lp_bytes`, `lp_text`, `nullable_lp_*`,
   `genesis_hash`, `row_digest_pg17_jsonb_text_sha256_v1`, `compute_event_hash`) — la **única**
   implementación del formato de bytes en todo el sistema; el verificador las invoca por SQL en
   vez de duplicarlas en Java, eliminando el riesgo de que dos implementaciones diverjan
   silenciosamente.
6. Función trigger `emit_mutation_event()`, `SECURITY DEFINER`, `search_path` cerrado, sin
   `WHEN OTHERS`, OLD/NEW del motor, `FOR UPDATE` sobre `stream_head` para serializar por
   `(tenant,domain)`.
7. Un bloque `DO` que **calcula** — no hardcodea por segunda vez — el `coverage_registry` desde
   `information_schema`/`pg_class` reales, clasifica cada tabla `stir.*` real en `COVERED` (40),
   `EXCLUDED_JUSTIFIED` (5, con justificación textual) o `NO_PRIVILEGE_CATALOG` (2, verificado en
   vivo con `has_table_privilege`), y **falla la migración** (`RAISE EXCEPTION`) si aparece una
   tabla sensible sin clasificar — el gate de CI que `CLAUDE_GOVERNED_STATE_AUDIT_ORDERS.md` exige,
   aplicado en el momento de migrar, no sólo en un test posterior.
8. Un segundo bloque `DO` que da a `stir_auditor` `SELECT` + una política RLS adicional
   (`USING (true)`) sobre cada una de las 40 tablas `COVERED`, sin tocar las políticas ni grants ya
   existentes de `idax_app`/`idax_admin`.

No se modificó ninguna migración existente (V1–V17), ningún grant/rol de IDAX Core/osTRIS/IDAX
Ledger, ni `idax_admin` (que sigue con cero DML STIR desde V17).

## Role/grant matrix (Fase 1)

| Rol | `stir.*` (45 tablas gobernadas) | `stir_audit.mutation_event` / `stream_head` | `stir_audit.coverage_registry` | `stir_audit.verifier_cursor` / `verification_result` | `stir_audit.security_incident` | `stir_audit.anchor_outbox` |
|---|---|---|---|---|---|---|
| `postgres` (migrador/superuser) | todo (infraestructura) | todo (bypassa RLS) | todo | todo | todo | todo |
| `stir_audit_owner` (NOLOGIN) | nada | dueño; sólo INSERT/UPDATE vía su propia política, nunca DELETE | dueño | nada (no lo necesita) | nada | nada |
| `stir_auditor` (LOGIN, secreto propio) | **SELECT únicamente**, cross-tenant (RLS propia) | **SELECT únicamente** | SELECT | SELECT/INSERT/UPDATE (sus propios veredictos/cursor) | SELECT/INSERT (nunca UPDATE/DELETE) | SELECT/INSERT/UPDATE (sin uso en Fase 1) |
| `idax_app` / `idax_admin` / `idax_backend` | sin cambios respecto a V17 (idax_admin: 0 DML; idax_app: 45/45 INSERT, subset UPDATE, como antes) | **nada** (ni USAGE del esquema) | nada | nada | nada | nada |

Verificado con roles reales, no leído del código: ver "Ataques ejecutados" más abajo.

## Cobertura table-by-table

| Dominio | Tablas cubiertas (trigger activo) |
|---|---|
| `CONSTITUTION` (9) | `market_constitution`, `constitutional_proposal`, `constitutional_signature`, `market_governance_event`, `constitutional_authority`, `constitutional_seat`, `constitutional_credential_history`, `constitutional_webauthn_credential`, `constitutional_webauthn_challenge` |
| `INTEGRITY` (2) | `market_integrity_case`, `market_integrity_case_event` |
| `ORDINARY` (8) | `community_governance_settings`, `community_governance_member`, `ordinary_governance_policy`, `ordinary_proposal`, `ordinary_proposal_electorate`, `ordinary_vote`, `ordinary_proposal_execution`, `community_seed` |
| `REFERENCE` (6) | `community_reference`, `reference_policy`, `reference_snapshot`, `reference_proposal`, `reference_context_snapshot`, `reference_definition` |
| `CONSENT_RETENTION` (5) | `reference_consent`, `reference_consent_event`, `retention_policy`, `retention_lifecycle_event`, `reference_observation` |
| `AGREEMENT_ECONOMIC` (7) | `agreement`, `agreement_snapshot`, `offer`, `negotiation`, `trade`, `marketplace_economic_binding`, `participant_economic_binding` |
| `MARKETPLACE_IDENTITY` (3) | `listing`, `listing_revision`, `participant_independence_projection` |

**Total COVERED: 40/47 tablas reales.**

| Excluida (justificada) | Motivo |
|---|---|
| `attachment` | Lifecycle de ficheros/fotos, no decisión de gobernanza; ya tiene su propio contrato operativo de DELETE. |
| `participant_device_credential` | Bookkeeping de conveniencia UX del navegador; la credencial constitucional real (`constitutional_webauthn_credential`) sí está cubierta. |
| `content_report` | Intake de moderación (una denuncia/claim), no una decisión de autoridad. |
| `notification` | Sin semántica de gobernanza. |
| `participant_profile` | Datos de perfil básicos, no superficie de decisión gobernada. |

| Catálogo (sin privilegio, verificado en vivo) | `category`, `resource_kind` — `idax_app` no tiene `INSERT` en ninguna hoy. |

**40 + 5 + 2 = 47 = total de tablas `stir.*` reales.** Ninguna tabla queda sin clasificar; el
propio bloque `DO` de la migración lo garantiza en tiempo de migrado, y
`coverageRegistryClassifiesEveryStirTable` (test, ver abajo) lo re-verifica contra una base viva.

## Verificador Java (`stir-backend/audit-verifier/`)

Proyecto Maven **completamente separado** (propio `pom.xml`, propio `Dockerfile`, sin `<modules>`
del reactor de `stir-backend`, sin dependencia del jar de `stir-backend`). Sin Spring Boot, sin
controladores STIR. Única dependencia STIR: una conexión JDBC autenticada como `stir_auditor`.

**[Actualizado tras la remediación — ver "Remediación de los 7 findings" arriba para el detalle
por finding.]**

- `Config` — lee `STIR_AUDITOR_*` únicamente; nunca `STIR_WEBAUTHN_*`/`STIR_JWT_*`/`OSTRIS_*`.
- `AuditSql` — toda consulta corre como `stir_auditor`; recomputa hash/digest llamando a las
  mismas funciones SQL de la migración (v1 o v2 según `event_format_version` de cada fila, nunca
  reinterpretando eventos v1 antiguos), nunca una segunda implementación en Java.
  `fetchUnverifiedEvents` (anti-join, P1-RA-001) reemplazó al antiguo `fetchEventsAfter` (watermark).
  `recordVerdictAtomically`/`recordChainIncidentAtomically` (P1-RA-002) escriben veredicto+incidente
  +cursor/checkpoint en una única transacción JDBC. `readConsistentTableSnapshot` (concurrencia,
  REPEATABLE READ) reemplazó dos lecturas autocommit separadas.
- `ChainVerifier` — continuidad de secuencia/hash por stream, agnóstica de dominio; ganó
  `reconcileStreamFrom`/`ReconciliationOutcome` (P1-RA-007) para el recorrido periódico completo
  desde génesis.
- `DomainRuleEngine` + `domain/*Rule` — una regla por dominio; `IntegrityRule` fue reescrita
  completa (P1-RA-004) para validar la cadena de transiciones, no sólo el primer estado.
- `Reconciler` — full-scan: filas vivas sin evento, filas cuyo último evento no es DELETE pero ya
  no existen; ahora excluye claves en `stir_audit.baseline_import` (P1-RA-003) y usa un snapshot
  MVCC consistente de una sola transacción (concurrencia).
- `HealthServer` — endpoint `/health` sin framework (`com.sun.net.httpserver`), sin UI.
- `VerifierLoop`/`Main` — polling durable, ahora con dos pasadas explícitas por ciclo: incremental
  (cada ciclo) + reconciliación periódica de cadena completa (cada 10 ciclos, P1-RA-007). El escritor
  de incidentes separado (`IncidentReporter`) se eliminó — su responsabilidad quedó absorbida dentro
  de las transacciones atómicas de `AuditSql`. `LISTEN`/`NOTIFY` sigue reservado como optimización
  futura (el trigger no hace `NOTIFY` todavía).

### Decisión de alcance explícita: nunca `PASS_CRYPTO` en Fase 1

Verificar de verdad una firma Ed25519/WebAuthn exige reimplementar independientemente RFC 8785 JCS
más la verificación de firma — el verificador **deliberadamente no importa**
`SevenKeysCrypto`/`WebAuthnCrypto` de `stir-backend` (eso convertiría al backend en autoridad
implícita de los resultados del verificador). Esa reimplementación independiente no se abordó en
el tiempo/alcance de la Fase 1. **Ningún dominio devuelve `PASS_CRYPTO` hoy** — el máximo posible es
`PASS_STRUCTURE_ONLY` (recuento de firmas/secuencia correcto, identidad del firmante no probada
criptográficamente). Esto es exactamente lo que `CLAUDE_GOVERNED_STATE_AUDIT_ORDERS.md` permite:
"si necesitas una firma/protocolo nuevo... no lo inventes. Deja SPEC GAP." Ver "Gaps" más abajo.

## Ataques ejecutados (contra roles reales, base real)

Todos reproducidos dos veces: manualmente vía `psql`/`SET ROLE` contra un PostgreSQL 17 aislado
(`stir-audit-slot01-pg`, contenedor Docker propio, red propia `stir-audit-slot01-net`, nunca la VM
DEV ni ningún checkout activo), y de forma automatizada en `VerifierDetectionTest` (Testcontainers,
12 tests, 0 fallos) más `StirAdminAuthorityPostgresTest`/`StirAdminAuthorityUpgradePostgresTest`
existentes (sin cambios de comportamiento, sólo el número de migración esperado).

| # | Ataque | Rol | Resultado |
|---|---|---|---|
| 1 | `INSERT` directo de `market_constitution` con `version=999999`, sin `constitutional_proposal` | `idax_app` | Evento atómico creado; `VerificationVerdict.VIOLATION`, `CONSTITUTION_WITHOUT_MATCHING_PROPOSAL`. **Éste es el ataque #1 de `AUD-012`.** |
| 2 | `INSERT` directo de `market_integrity_case_event(status='FINAL')` sin `SIGNAL`/`UNDER_REVIEW` previos | `idax_app` | Evento atómico creado; `VIOLATION`. Razón actualizada tras P1-RA-004: `FIRST_EVENT_NOT_SIGNAL` (antes `FINAL_WITHOUT_PRIOR_HISTORY`; la regla ahora valida la cadena completa, no un caso especial). **Ataque #2 de `AUD-012`.** |
| 3 | `INSERT` en `stir_audit.mutation_event` | `idax_app`, `idax_admin` | `permission denied for schema stir_audit` (ni siquiera llega a la tabla — falta `USAGE`) |
| 4 | `DELETE` en `stir_audit.mutation_event` | `idax_app`, `idax_admin` | denegado |
| 5 | `UPDATE` de `stir_audit.stream_head` | `idax_app`, `idax_admin` | denegado |
| 6 | `TRUNCATE stir_audit.mutation_event` | `idax_app`, `idax_admin` | denegado |
| 7 | `DROP FUNCTION stir_audit.emit_mutation_event()` | `idax_app`, `idax_admin` | denegado (`must be owner`) |
| 8 | `ALTER TABLE ... DISABLE TRIGGER` | `idax_app`, `idax_admin` | denegado (`must be owner`) |
| 9 | `SET session_replication_role = replica` | `idax_app`, `idax_admin`, `idax_backend` | denegado (`permission denied to set parameter`) |
| 10 | Manipular `current_hash` almacenado (simulando owner/superuser comprometido, **fuera del modelo de amenaza** pero probado igualmente) | `postgres` (owner) | El propio `ChainVerifier` lo detecta: `STORED_HASH_MISMATCH` — la matemática de la cadena, no el límite de rol, atrapa esto |
| 11 | Rollback de una mutación gobernada | `idax_app` | Cero eventos, cero avance de `stream_head` — el trigger revierte con la transacción |
| 12 | `INSERT` multirow (3 filas, una sentencia) | `idax_app` | 3 eventos, secuencia 1,2,3 contigua |

## Resultados del verifier

**[Actualizado tras la remediación.]**

- `VerifierDetectionTest`: **29/29 PASS** (Testcontainers PostgreSQL 17, migraciones reales V1–V19;
  era 12/12 antes de la remediación — los 17 tests nuevos son la evidencia de regresión de
  P1-RA-001...007, ver sección de remediación arriba para el detalle por test).
- `StirAuditCoveragePostgresTest` (nuevo, reactor `stir-backend`, P1-RA-006): **3/3 PASS**.
- Enmienda constitucional legítima (7 firmas simuladas, sin criptografía real) → `PASS_STRUCTURE_ONLY`,
  nunca `PASS_AUTHORIZED` (aserción explícita en el test).
- Cadena MI legítima SIGNAL→UNDER_REVIEW→FINAL con decisor≠originador → `PASS_STRUCTURE_ONLY`.
- Aislamiento de tenant: una mutación en tenant A nunca crea/avanza `stream_head` de tenant B.
- `coverage_registry` cubre el 100% de `pg_tables('stir')` real, verificado ahora en **dos**
  reactores Maven independientes (audit-verifier y stir-backend), no sólo uno — ver P1-RA-006.

## Restart real del proceso (no sólo idempotencia simulada)

Checkpoint original (primera entrega): con 5 ataques ya cometidos en una base aislada, se arrancó
el jar empaquetado (`stir-audit-verifier.jar`), procesó y reportó los 5 incidentes, se mató con
`SIGKILL` a mitad de ciclo, y se reinició. `select count(*) from stir_audit.security_incident` = **5
antes y 5 después**.

Remediación (segunda entrega): `VerifierDetectionTest.verifierLoopRestartProcessesRemainingEventsExactlyOnce`
automatiza el mismo tipo de prueba contra el punto de entrada real de producción
(`VerifierLoop.runOneIncrementalPassForTest`) — una instancia nueva de `VerifierLoop`+`AuditSql`
("proceso 2", simulando el reinicio tras un crash justo después del pase del "proceso 1") vuelve a
procesar el mismo stream y confirma cero filas `verification_result` duplicadas. La prueba manual
`SIGKILL` real del jar empaquetado no se repitió en esta remediación (el mecanismo de idempotencia
que la sostiene — `ON CONFLICT` + `dedup_key` — es exactamente lo que P1-RA-002 endureció y lo que
`failureMidTransactionLeavesNoPartialVerdictOrIncident`/`reprocessingAfterSimulatedCrashProducesNoDuplicateIncident`
prueban directamente).

## Clean install / upgrade

**[Actualizado tras la remediación.]**

- PostgreSQL 17 limpio, V1→V19 vía Flyway real: **PASS** (cada ejecución de `VerifierDetectionTest`/
  `StirAuditCoveragePostgresTest` lo repite desde cero contra Testcontainers).
- V16/V17→V19 con historia preexistente: **PASS** —
  `StirAdminAuthorityUpgradePostgresTest` migró de V16/V17 a V19 sin fallos (`"Successfully applied
  3 migrations to schema stir, now at version v19"` en la corrida de este checkpoint), historia
  preservada, `idax_admin` sigue con cero DML tras la migración completa.
- V18→V19 con datos legacy reales pre-baseline (el caso específico que P1-RA-003 exige): **PASS** —
  ver `baselineImportSuppressesLegacyRowsButNotGenuinelyOrphanedOnes` en la sección de remediación.
- `mvn verify` en `stir-backend`: **227/227 tests, 0 fallos** (era 224/224 antes de la remediación;
  los 3 tests nuevos son `StirAuditCoveragePostgresTest`, P1-RA-006). Ejecutado en esta misma sesión
  de remediación, tras aplicar V19 completa.
- `mvn test` en `audit-verifier`: **29/29 tests, 0 fallos** (era 12/12).
- `docker compose -p stir-audit-remediation-slot01 config --quiet`: **PASS**, re-ejecutado en esta
  remediación (proyecto Compose aislado con nombre nuevo, nunca `stir-dev`; ningún contenedor de
  `stir-dev` tocado, verificado por nombre de proyecto antes y después).
- `docker compose -p stir-audit-remediation-slot01 build audit-provision stir-audit-verifier`:
  **PASS**, re-construido con el código Java remediado — el `stir-audit-verifier.jar` de esta imagen
  incluye los fixes de P1-RA-001/002/005/007 (Maven shade dentro del propio build de la imagen, no
  un jar cacheado de antes de la remediación). Sólo se construyeron imágenes; no se arrancó ningún
  contenedor, por lo que no hay nada que detener.
- **No se levantó el stack completo** (los cuatro vendors IDAX Core/Shell/osTRIS/Ledger exigirían
  reconstruir sus imágenes también, consumo de tiempo/recursos desproporcionado dado que el
  mecanismo de auditoría ya está probado exhaustivamente contra PostgreSQL real, incluyendo ahora
  concurrencia real con hilos JDBC). Pendiente para quien retome: `docker compose -p
  <nombre-aislado> up -d --build` completo en un slot con acceso a los cuatro vendors, nunca en la
  VM DEV activa.

## Mediciones de rendimiento (medidas, no estimadas)

500 `INSERT` secuenciales en `stir.reference_definition`, mismo tenant/dominio (peor caso de
contención: todas serializan sobre el mismo lock de `stream_head`), un solo conector, PostgreSQL 17
en contenedor Docker local:

| Métrica | Sin trigger | Con trigger (V18) |
|---|---|---|
| p50 | 34 µs | 1526 µs |
| p95 | 97 µs | 2455 µs |
| p99 | 244 µs | 3121 µs |
| media | 58 µs | 1710 µs |
| máximo observado | — | 27100 µs |

Overhead añadido: **~1.5 ms p50 por escritura gobernada**, bajo el patrón de contención más
desfavorable (escritor único, mismo stream). Tamaño medido: **~933 bytes por evento** (tabla +
índices, 500 eventos = 456 kB). No se midió throughput con múltiples escritores concurrentes en el
mismo stream ni en streams distintos — pendiente si el volumen de piloto lo justifica.

## Findings nuevos (durante la implementación de este MVP, no en el sistema previo)

Ninguno de estos afecta a `stir.main` real ni a la VM DEV — todos se encontraron y arreglaron
dentro de este mismo worktree, antes de que el código saliera de él:

1. Las funciones auxiliares de canonicalización se crearon inicialmente como `postgres`
   (`RESET ROLE` prematuro) en vez de como `stir_audit_owner` — la función `SECURITY DEFINER`
   no tenía permiso para invocarlas. Corregido moviendo el `SET ROLE stir_audit_owner` para cubrir
   toda la sección.
2. Faltaban los `GRANT` de tabla para `stir_auditor` (RLS por sí sola no basta en PostgreSQL —
   necesita también el grant de tabla). Encontrado ejecutando de verdad el jar empaquetado contra
   la base real, no por inspección.
3. Faltaba `GRANT USAGE ON SCHEMA stir TO stir_auditor` — mismo patrón, esquema aparte de tablas.
4. `U64(sequence)` inicialmente codificado con una concatenación de texto + cast a `bit(64)`, que
   no es sintaxis PostgreSQL válida para un literal hexadecimal; corregido con `int8send()` nativo.

## Gaps (SPEC GAP, no implementados por decisión explícita de alcance)

- **`PASS_CRYPTO` no implementado.** Ver "Decisión de alcance" arriba. Para cerrarlo: reimplementar
  independientemente verificación Ed25519/WebAuthn + canonicalización JCS en el verificador, con
  sus propios vectores de test, sin importar el código de `stir-backend`.
- **External anchor (IDAX Ledger) — sólo interfaz, cero implementación.** `anchor_outbox` existe
  como tabla; ningún código la escribe. Explícitamente diferido a Fase 2 por instrucción directa.
- **Consumption gate (verificación antes de uso) — no implementado.** `ReferenceService` sigue
  consumiendo `market_constitution`/`market_integrity_case_event` de forma síncrona, sin esperar al
  veredicto del verificador. La ventana entre el `INSERT` fraudulento y la detección del verificador
  (ciclo de polling, ~2 s por defecto) sigue siendo real y sin cerrar.
- **`LISTEN`/`NOTIFY` no emitido por el trigger** — reservado como optimización de latencia futura;
  el polling durable es el mecanismo real hoy.
- ~~Reconciliación bajo concurrencia real no probada~~ — **RESUELTO en la remediación** (P1-RA-003
  bug de concurrencia notado por Codex): `AuditSql.readConsistentTableSnapshot` usa
  `REPEATABLE READ, READ ONLY` para las tres lecturas de `Reconciler`; probado con escritores JDBC
  concurrentes reales, no sólo argumentado. Ver sección de remediación, "Concurrencia".
- **Nuevo coste documentado por la remediación de P1-RA-007**: la reconciliación periódica de cadena
  completa ahora recorre CADA stream desde génesis en cada pase periódico (cada 10 ciclos por
  defecto), no sólo desde el último checkpoint — necesario para no reabrir el hueco que Codex
  encontró (ver "P1-RA-007" arriba). Coste O(longitud de la cadena) por pase periódico y por stream;
  no medido a escala de producción/piloto en este checkpoint. Si el volumen de un piloto real lo
  justifica, una Fase 2 podría necesitar un esquema de verificación incremental con compromisos
  intermedios (p.ej. Merkle checkpoints firmados) en vez de recorrer siempre desde génesis — fuera
  de alcance de esta remediación, no inventado aquí.
- **Restart/kill del propio proceso PostgreSQL** (no sólo del verificador) no probado en este
  checkpoint ni en la remediación.
- **Prueba manual `SIGKILL` real del jar empaquetado** no repetida en esta remediación (ver
  "Restart real del proceso" arriba) — cubierta en su lugar por un test automatizado equivalente
  contra el punto de entrada de producción real.

## Estado exacto de `AUD-012`

**HIGH, abierto. Sin cambios de severidad.** Este MVP demuestra detección tras el hecho para los
dos ataques concretos de `AUD-012` y para manipulación de la propia cadena de auditoría — no
demuestra prevención, no prueba intención/identidad del actor más allá de una estructura
autoconsistente, y no cierra la ventana entre escritura fraudulenta y verificación. Por instrucción
explícita del usuario, **el objetivo de este MVP nunca fue cerrar `AUD-012`** — era demostrar que la
opción B (trigger + cadena + verificador independiente) es técnicamente viable sin exagerar lo que
prueba. Se cumple.

## No se ha hecho

- Merge ni push a `main` en ningún repo.
- Despliegue en la VM DEV activa ni en ningún checkout compartido.
- Cambios a Seven Keys, 7-of-7, Guardian, Ordinary Governance, osTRIS, FFM, Community Value
  References.
- Anclaje IDAX Ledger real (ni siquiera en modo de prueba controlada).
- Verificación antes de consumo.
- Fase 2 en ninguna forma.

## Siguiente paso

Tercera reauditoría adversarial independiente de Codex sobre esta entrega remediada. La primera
reauditoría cerró (con remediación) P1-RA-001...007; la segunda reauditoría encontró dos hallazgos
nuevos bloqueantes sobre esa misma remediación (P1-R2-001, falso CRITICAL en Market Integrity
legítimo; P1-R2-002, semántica de baseline exagerada) más una carrera no cerrada en `ChainVerifier`
y un defecto operativo de deduplicación de alertas - los cuatro se abordan en "## Remediación de la
segunda reauditoría" arriba. Dictamen esperado de la tercera reauditoría: estos cuatro puntos
aceptados como remediados, o nuevos findings sobre el propio código de esta segunda remediación.
Sólo después de esa tercera reauditoría corresponde decidir: anclaje externo, consumption gate,
cambios preventivos adicionales, o inicio de Fase 2 — per `CLAUDE_GOVERNED_STATE_AUDIT_ORDERS.md`.
Ningún merge/push a `main` ni despliegue en DEV activa hasta entonces.
