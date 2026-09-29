# Tamper-Evident Governed State Audit MVP — Fase 1: validación y checkpoint

Estado: **Fase 1 completa, entregada para revisión de Codex. Fase 2 NO iniciada.** `AUD-012`
permanece **HIGH, abierto**. Este documento es el checkpoint solicitado en
`CLAUDE_GOVERNED_STATE_AUDIT_ORDERS.md`: no se ha hecho merge ni push a `main` en ningún repo, ni
desplegado en la VM DEV activa. Todo el trabajo vive en worktrees aislados bajo
`stir/.local/full-system-audit/slot-01/`.

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

- `Config` — lee `STIR_AUDITOR_*` únicamente; nunca `STIR_WEBAUTHN_*`/`STIR_JWT_*`/`OSTRIS_*`.
- `AuditSql` — toda consulta corre como `stir_auditor`; recomputa hash/digest llamando a las
  mismas funciones SQL de la migración (una sola implementación, no dos).
- `ChainVerifier` — continuidad de secuencia/hash por stream, agnóstica de dominio.
- `DomainRuleEngine` + `domain/*Rule` — una regla por dominio, ver "Decisión de alcance" abajo.
- `Reconciler` — full-scan: filas vivas sin evento, filas cuyo último evento no es DELETE pero ya
  no existen.
- `IncidentReporter` — único escritor de `security_incident`; único emisor del log `CRITICAL`.
- `HealthServer` — endpoint `/health` sin framework (`com.sun.net.httpserver`), sin UI.
- `VerifierLoop`/`Main` — polling durable con cursor, `LISTEN` reservado como optimización futura
  (el trigger no hace `NOTIFY` todavía).

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
| 2 | `INSERT` directo de `market_integrity_case_event(status='FINAL')` sin `SIGNAL`/`UNDER_REVIEW` previos | `idax_app` | Evento atómico creado; `VIOLATION`, `FINAL_WITHOUT_PRIOR_HISTORY`. **Ataque #2 de `AUD-012`.** |
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

- `VerifierDetectionTest`: **12/12 PASS** (Testcontainers PostgreSQL 17, migraciones reales V1–V18).
- Enmienda constitucional legítima (7 firmas simuladas, sin criptografía real) → `PASS_STRUCTURE_ONLY`,
  nunca `PASS_AUTHORIZED` (aserción explícita en el test).
- Cadena MI legítima SIGNAL→UNDER_REVIEW→FINAL con decisor≠originador → `PASS_STRUCTURE_ONLY`.
- Aislamiento de tenant: una mutación en tenant A nunca crea/avanza `stream_head` de tenant B.
- `coverage_registry` cubre el 100% de `pg_tables('stir')` real (test dedicado).

## Restart real del proceso (no sólo idempotencia simulada)

Con 5 ataques ya cometidos en una base aislada: se arrancó el jar empaquetado
(`stir-audit-verifier.jar`), procesó y reportó los 5 incidentes, se mató con `SIGKILL` a mitad de
ciclo, y se reinició. `select count(*) from stir_audit.security_incident` = **5 antes y 5
después** — el cursor durable (`stir_audit.verifier_cursor`) evitó tanto perder progreso como
duplicar incidentes.

## Clean install / upgrade

- PostgreSQL 17 limpio, V1→V18 vía Flyway real: **PASS** (repetido más de diez veces durante el
  desarrollo de este checkpoint).
- V16→V18 con historia preexistente (una fila insertada por `idax_admin` antes del fix de V17,
  exactamente como `StirAdminAuthorityUpgradePostgresTest` ya probaba para V16→V17): **PASS**,
  historia preservada, `idax_admin` sigue con cero DML tras la migración completa.
- `mvn -s .mvn/public-settings.xml clean verify` en `stir-backend`: **224/224 tests, 0 fallos** (sin
  regresión respecto al baseline pre-Fase-1; el único cambio necesario fue actualizar
  `StirAdminAuthorityUpgradePostgresTest` para esperar `>= 18` en vez del literal `"17"`).
- `docker compose -p stir-audit-slot01 config --quiet`: **PASS** (sintaxis válida, proyecto Compose
  aislado, nunca `stir-dev`).
- `docker compose -p stir-audit-slot01 build audit-provision stir-audit-verifier`: **PASS**, imagen
  construida.
- **No se levantó el stack completo** (los cuatro vendors IDAX Core/Shell/osTRIS/Ledger exigirían
  reconstruir sus imágenes también, consumo de tiempo/recursos desproporcionado dado que el
  mecanismo de auditoría ya está probado exhaustivamente contra PostgreSQL real). Pendiente para
  quien retome: `docker compose -p <nombre-aislado> up -d --build` completo en un slot con acceso
  a los cuatro vendors, nunca en la VM DEV activa.

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
- **Reconciliación bajo concurrencia real no probada** — el full-scan del `Reconciler` no toma un
  snapshot MVCC coordinado; un commit que llega durante el propio scan podría producir un falso
  incidente transitorio. Documentado, no resuelto.
- **Restart/kill del propio proceso PostgreSQL** (no sólo del verificador) no probado en este
  checkpoint.

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

Revisión de Codex sobre este checkpoint antes de decidir: anclaje externo, consumption gate,
cambios preventivos adicionales, o inicio de Fase 2 — per `CLAUDE_GOVERNED_STATE_AUDIT_ORDERS.md`.
