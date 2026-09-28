# Full system + DEV VM audit — STIR/osTRIS/IDAX

Estado: **EN CURSO**, iniciado 2026-09-28. Auditoría adversarial de `main`, sin nuevas funcionalidades ni cambios de producción. Ningún dictamen de piloto se deduce de tests que sólo cubren localhost.

## Baseline congelado antes de escrituras

Los cinco repos STIR de PC estaban en `main` y limpios (`git status --porcelain=v1` vacío):

| Repositorio | HEAD | Remote |
|---|---|---|
| `stir` / stir-workspace | `878b59a9ab9dd05078c2c100f4f1474c07dd36dc` | `https://github.com/toni-soler/stir-workspace.git` |
| `stir-backend` | `e127f20f8d71224fe74c05832e8ed91db79eaa40` | `https://github.com/toni-soler/stir-backend.git` |
| `stir-frontend` | `9fb22115ae15122252a02c5e3300d9059520ac2d` | `https://github.com/toni-soler/stir-frontend.git` |
| `stir-main` | `bbfbba4922908d35b62636da951c9b50e33805b8` | `https://github.com/toni-soler/stir-main.git` |
| `stir-doc` | `cdfdabdfb08c6ef7be74cccae6378ef60146d725` | `https://github.com/toni-soler/stir-doc.git` |

`stir-main/upstream.lock.json` fija IDAX Core runtime `2b34369b`, Shell `a44141d5`, IDAX Ledger `ca177144` y osTRIS `1cd7e53e`; sus fuentes vendorizadas reales están bajo `stir-main/vendor/`. Los checkouts vecinos `idax/idax-platform` y `ostris` estaban limpios; `idax/` tenía un reporte generado modificado y `ostris-doc/` un directorio sin seguimiento. No se tocarán estos dos checkouts contaminados. El repositorio `idax-ledger/` local estaba limpio, pero en `wip/ledger-permission-catalog`, por lo que **no** se asume que sea el código del Ledger desplegado: el pin vendorizado es la fuente que hay que comparar.

Los cuatro vendor HEAD coinciden exactamente con `upstream.lock.json`. Core y Shell están limpios; los worktrees vendor de osTRIS (12 paths staged) y Ledger (1 path staged) **no** están limpios porque `scripts/prepare_shell.py` aplica tres patches públicos versionados en `stir-main/patches/`. `git apply --check --reverse` pasó para cada patch; no hay diff unstaged ni archivos untracked. Esta es una excepción de baseline prevista por el build, no trabajo de otro agente; se auditó el código vendorizado **más** esos patches y no se modificó ningún vendor. La VM debe confirmar si ejecuta el mismo material preparado.

PC: Docker client/server 27.0.3; Java 25.0.1; Node 20.16.0; Python 3.14.3. La composición local `stir-dev` apareció al primer inventario, con imágenes `stir` `sha256:dd004f56`, Shell `sha256:1c27a408`, osTRIS `sha256:45b503b8`, Ledger `sha256:f13f700a`, y volúmenes `stir-dev_postgres_data` y `stir-dev_minio_data`. Antes de poder consultar sus migraciones los contenedores desaparecieron de Docker Desktop por una operación externa a esta auditoría; los volúmenes permanecieron. No se reinició ni alteró esa composición. Esta discrepancia impide usarla como baseline estable de DEV.

Version surface local: backend Maven `0.5.0-rc1` (Spring Boot parent `3.4.4`), frontend npm `0.5.0-rc1`; `stir-main` pin por commit arriba. El contrato público de DEV informa `stirVersion=0.5.0-rc1`, `catalogContractVersion=1` y `externalContractSchemaVersion=3`, pero aún no identifica los HEAD realmente desplegados. El navegador integrado usado para la prueba pública fue Codex In-app Browser; su versión de motor no se ha inventariado aún.

La VM DEV está documentada como `dev.stir.es`, con `openresty` terminando TLS. `https://dev.stir.es/api/stir/instance` respondió HTTP 200 el 2026-09-28 con `stirVersion=0.5.0-rc1`, catálogo 1 y snapshot externo 3; `/actuator/health/readiness` respondió 200 con `X-Correlation-Id`/`X-Request-Id`. El primer intento al antiguo `192.168.1.125:22` fue rechazado. El usuario proporcionó el acceso vigente `ssh -p 12522 stir-admin@2.139.185.159`, verificado después. La URL pública anuncia `publicBaseUrl=http://localhost:8089` pese al origen HTTPS: `AUD-001` queda confirmado en el despliegue.

### Inventario DEV recuperado después del HIGH gate (2026-09-28 22:15 UTC)

- Los cinco repos bajo `/srv/stir/stir` están en `main` con los HEAD del baseline previo: workspace `878b59a9ab9d`, backend `e127f20f8d71`, frontend `9fb22115ae15`, main `bbfbba492290`, doc `cdfdabdfb08c`. Workspace, backend, frontend y doc están limpios. `stir-main` tiene `.venv/` no versionado, creado durante setup el 28-09; su propiedad respecto a otra sesión está pendiente de aclaración. Por la regla de contaminación, **no se ha modificado ningún repositorio VM** ni desplegado el HIGH gate.
- Contenedores en marcha: UI STIR `7770140301bc`, backend `504523597774`, proxy `caddy:2.9-alpine`, Shell `0de771048600`, Ledger `8f2aeb5049d7`, osTRIS `7b2301780c56`, UI osTRIS `a67121a91067`, UI Ledger `85e1c4c164a2`, PostgreSQL `postgres:17-alpine`, MinIO `b6bfe7239bfc` (`pgsty/minio` por digest). Backend, proxy, Shell, Ledger, osTRIS, PostgreSQL y MinIO informan `healthy`; las dos UI no publican health. Volúmenes `stir-dev_postgres_data` y `stir-dev_minio_data`.
- Migraciones instaladas: STIR V16, osTRIS V4, IDAX Ledger V3, Shell V1, IDAX Core V91. PostgreSQL 17.11; Docker 29.8.1, Java 21.0.12.1, Node 22.23.3, Python 3.12.3. La VM conserva 61 GB libres de 95 GB en `/srv/stir`. El esquema STIR de DEV sigue PRE-FIX: el rol real `idax_admin` conserva `INSERT` en `market_constitution`, `market_integrity_case`, `market_integrity_case_event`, `ordinary_vote`, `reference_consent_event` y `reference_policy`. Esto confirma la exposición del despliegue antiguo, no invalida la revalidación local V17.
- Checkpoint fuera de repositorios: `/srv/stir/audit-checkpoints/20260928T2211Z-pre-v17/postgres.dump` (`pg_dump` custom, 1.045.704 bytes, SHA-256 `46d899113852dfc0f4edc78719b9ea0c9160b1a9b66b5ce61f55ea8c3f25b616`) y `minio_data.tar.gz` (9.017 bytes, SHA-256 `d373174fb002359854c8d5a91922d62925c658dda1d1165d377e968208d2e3a0`). `pg_restore -l` y `tar -tzf` pasaron; directorio `0700`, archivos `0600`. Es un respaldo secuencial de servicios activos, **no** una prueba de restauración ni snapshot atómico. No se ha restaurado sobre la única copia DEV.

## Evidencia ejecutada sobre `main` sin modificaciones

- `mvn -q verify` en `stir-backend`: 216 tests, 0 fallos/errores/omitidos. Testcontainers PostgreSQL 17.11 aplicó V1–V16 desde esquema `stir` vacío en varias instancias. Esto no prueba la composición entera ni la base real de VM.
- `npm test` en `stir-frontend`: 45 tests, 0 fallos. `npm run build` y `npm run i18n:validate`: PASS, 12 locales y 560 claves.
- `python scripts/audit-public.py` en `stir-main`: PASS para 327 ficheros STIR propios. Es una búsqueda acotada, no un escáner completo de secretos/dependencias.
- HTTPS DEV: certificado aceptado por `curl`, proxy `openresty`, HSTS, CSP, `X-Content-Type-Options`, `X-Frame-Options` presentes. Falta sesión, upload, WebAuthn y recorrido de navegador autenticado.
- En el navegador integrado, `https://dev.stir.es/` carga la pantalla de login de IDAX Shell sobre HTTPS. El intento de abrir «Explorar la interfaz» dejó el foco en email con validación «Completa este campo» incluso tras recarga; el botón DOM es `type=button` en el pin de Shell, así que la causa queda **inconclusa** y no se atribuye aún a STIR. Ningún flujo autenticado se ha verificado.
- Sondas anónimas HTTPS: `/api/shell/v1/extensions`, `/api/shell/v1/platform/tenants` y `/actuator/env` devolvieron 403; un listing con tenant UUID nulo devolvió 401. Esto sólo verifica denegación anónima de esas rutas, no autorización entre usuarios ni tenants.
- Revisión dirigida de la suite existente: `OrdinaryGovernancePostgresTest` cubre snapshot electoral, quorum, voto único, SuperAdmin, stale policy y ejecución doble; `SevenKeysPostgresTest` cubre 7-of-7, replay de firma, freeze, Guardian y RLS; `WebAuthnSevenKeysPostgresTest` cubre replay/cross-proposal/cross-community/cross-tenant y contador. Todas corrieron en el `mvn verify` inicial; su alcance sigue siendo PostgreSQL/Testcontainers y claves simuladas, no ceremonia VM con browser/authenticator.
- Búsqueda estática en `stir-backend/src/main/java` y migraciones V1–V16 por `fiatValue`, `exchangeRate`, `EUR/OST`, `redemption`, `convertToFiat`, `automatic conversion` y `fiscal valuation`: 0 coincidencias. Es una comprobación acotada del código STIR, no prueba universal de ausencia de paridad en docs, vendor ni runtime.
- Búsqueda ampliada de paridad/conversión en los cinco repos STIR y `stir-main/vendor/{ostris,idax-ledger}` (excluidos `node_modules`, `target`, `dist`): sin coincidencias de código para `exchangeRate`, `fiatValue`, `convertToFiat`, `redemption.value`, `fiat.peg` ni `EUR/OST`; el único match documental de esa búsqueda es la prohibición explícita de paridad en `MIXED_CONSIDERATION_EXTENSION.md`. Esta búsqueda no demuestra por sí sola todas las semánticas posibles de reporting/DAC7.
- En código STIR `OstrisClient.propose()` fija exclusivamente `purpose=EXCHANGE`, entradas con cuentas pagadora/receptora y `contractualMetadataDigest` del Agreement; `TradeService` deriva importe del Offer aceptado y consulta a osTRIS para `COMMITTED`. No hay llamada STIR a `SETTLEMENT` ni a una operación `forceTransaction` en el cliente actual. La auditoría runtime de todas las llamadas y firmas queda pendiente de VM.

## Findings pre-fix

### AUD-001 — MEDIUM — URL pública DEV anuncia localhost

- **Area:** despliegue HTTPS / contratos de instancia.
- **Invariant:** la URL pública anunciada debe corresponder al origen HTTPS efectivo de DEV.
- **Reproduction:** `curl.exe https://dev.stir.es/api/stir/instance`.
- **Observed:** HTTP 200 desde `https://dev.stir.es`, pero `publicBaseUrl` es `http://localhost:8089`.
- **Expected:** `https://dev.stir.es` o una URL pública DEV equivalente.
- **Impact:** consumidores que construyan enlaces absolutos, invitaciones o callbacks desde este contrato pueden enviar al usuario a su propio localhost; la exactitud del version surface externo queda comprometida. No demuestra por sí solo fallo de WebAuthn, cuyo RP ID/origin se configuran aparte.
- **Evidence:** respuesta JSON DEV del 2026-09-28; `stir-backend/application.yml` usa ese localhost como default y `stir-main/compose.yml` lo propaga si falta `STIR_PUBLIC_BASE_URL`.
- **Root cause:** override DEV ausente o no efectivo; confirmar en VM cuando haya acceso.
- **Recommended remediation:** fijar `STIR_PUBLIC_BASE_URL=https://dev.stir.es` en configuración DEV, reiniciar sólo el servicio pertinente tras checkpoint, comprobar contrato y flujos que lo consumen.
- **Status:** FOUND; sin fix.
- **Regression test:** GET HTTPS público debe anunciar el mismo origen externo configurado y ninguna URL loopback.

### AUD-002 — SPEC GAP — hold de integridad perpetuo tras DISMISSED

- **Area:** consentimiento/retención e integridad de mercado.
- **Invariant:** una razón de conservación debe tener semántica y finalización explícitas; un caso cerrado/desestimado no debe convertirse accidentalmente en retención indefinida de identificadores.
- **Reproduction:** rama aislada backend `codex/full-system-audit-repro`, commit `ea1ab27`, test `ConsentRetentionPostgresTest#auditReproductionDismissedCaseStillHoldsIdentifiersAfterRetentionWindow`: observación de 200 días; caso SIGNAL → UNDER_REVIEW → DISMISSED por otro actor; consultar retención y probar anonimización. `mvn -q -Dtest=ConsentRetentionPostgresTest#auditReproductionDismissedCaseStillHoldsIdentifiersAfterRetentionWindow test` pasó en PostgreSQL 17.11 limpio con V1–V16.
- **Observed:** último evento `DISMISSED`; `statusFor()` devuelve `MUST_BE_RETAINED`, no aparece en due list y `anonymize()` rechaza. El código comprueba mera existencia del caso sin consultar el último estado.
- **Expected:** regla documentada de cuándo termina el hold, que preserve evidencia e historial sin conservar identificadores innecesariamente.
- **Impact:** posible retención indefinida de identificadores personales. La severidad técnica se asignará tras reproducción y definición normativa; no se corregirá inventando una expiración.
- **Evidence:** `RetentionService.java` `statusFor()`, test de reproducción `ea1ab27`; la suite anterior sólo cubría un caso abierto.
- **Root cause:** criterio `hasCase` basado en existencia, no en estado/finalidad vigente.
- **Recommended remediation:** decisión legal/producto de lifecycle para `FINAL` y `DISMISSED`, seguida de test y cambio acotado si procede.
- **Status:** FOUND; sin fix.
- **Regression test:** estado de retención después de ventana vencida para OPEN, FINAL y DISMISSED, además de reconstrucción histórica. El test `ea1ab27` fija el comportamiento pre-fix, no una semántica nueva.

### AUD-004 — HIGH — `idax_admin` puede insertar constitución no firmada

- **Area:** privilegios PostgreSQL / Seven Keys.
- **Invariant:** la administración de plataforma no puede introducir una constitución ni modificar límites sin 7-of-7.
- **Reproduction:** rama aislada `codex/full-system-audit-repro`, commit `d9c8946`, test `ConsentRetentionPostgresTest#auditReproductionAdminDbRoleCanInsertUnsignedConstitution` en PostgreSQL 17.11 con V1–V16: crear definición, `SET LOCAL ROLE idax_admin`, insertar `market_constitution` con `constitutionalThreshold=1`, `provenanceRequired=false`, `independenceChecksRequired=false`, `concentrationChecksRequired=false`, volver a `idax_app` y leer snapshot. Ejecutado con `mvn -q -Dtest=ConsentRetentionPostgresTest#auditReproductionAdminDbRoleCanInsertUnsignedConstitution test`: PASS.
- **Observed:** INSERT administrativo admitido sin firma, authority real ni propuesta; el snapshot usa el digest/version de la constitución fabricada. V6 concede INSERT; V9 sólo revoca UPDATE de ciertas columnas; V16 no revoca el INSERT.
- **Expected:** sólo el flujo constitucional autorizado debe poder crear una nueva versión; el rol administrativo no debe tener mutación SQL equivalente.
- **Impact:** cualquier ruta con SQL como `idax_admin` puede introducir una constitución nueva y alterar los límites de referencia sin 7-of-7. **No** se ha demostrado explotación HTTP por un usuario SuperAdmin; esto es un bypass del límite de rol de DB, no una afirmación de control remoto.
- **Evidence:** migraciones V6/V9/V16, `ReferenceService.constitution()`, test `d9c8946` y su reporte Surefire (1 test, 0 fallos).
- **Root cause:** revocación incompleta de privilegios de mutación en V9.
- **Recommended remediation:** revocar INSERT administrativo en tablas constitucionales tras revisar todas las concesiones y dependencias del bootstrap; añadir una prueba de DB role que exija rechazo y verificar clean install + upgrade. No cambiar el protocolo ni 7-of-7.
- **Status:** FOUND → FIXED (`57cf4d5`, rama backend `codex/audit-high-gate`) → REVALIDATED en PostgreSQL 17.11 local y stack Docker aislado; integrado en `stir-backend/main` (`ea48c08`), VM DEV pendiente.
- **Regression test:** `StirAdminAuthorityPostgresTest` prueba roles y denegaciones directas en todas las tablas STIR; `StirAdminAuthorityUpgradePostgresTest` prueba V16→V17 conservando historia. E2E Seven Keys/WebAuthn comprueba que el servicio legítimo sigue operando.

**Post-fix gate (sin alterar el finding PRE-FIX):** V17 en `codex/audit-high-gate` revoca todo DML de `idax_admin` sobre tablas STIR, incluyendo privilegios por columna que un `REVOKE` de tabla no elimina. `StirAdminAuthorityPostgresTest` comprueba todos los objetos del esquema y una muestra de INSERT directos, no sólo `market_constitution`; `StirAdminAuthorityUpgradePostgresTest` prueba V16→V17 con historia existente. En el stack Docker aislado, el rol real `idax_admin` devolvió `permission denied for table market_constitution`, y el recuento de tablas STIR con DML administrativo fue cero. La suite Seven Keys/WebAuthn E2E siguió pasando tras V17. **Trust boundary:** el propietario/Flyway y el superuser de PostgreSQL conservan poder infraestructural; `idax_app` es el principal privado del servicio de dominio y sigue teniendo las escrituras necesarias para sus rutas verificadas. La posesión de la credencial de backend o ejecución SQL arbitraria como `idax_app` no se presenta como autoridad comunitaria legítima ni como algo que esta revocación pueda aislar. Hay que mantener esa credencial fuera de extensiones y usuarios.

### AUD-005 — LOW — documentación STIR contradice la norma osTRIS vendorizada

- **Area:** contrato de `TransactionPurpose.SETTLEMENT`.
- **Invariant:** la documentación de integración debe describir el protocolo realmente fijado, sin inventar ni negar vocabulario normativo.
- **Reproduction:** comparar `stir-doc/MIXED_CONSIDERATION_EXTENSION.md` con `stir-main/vendor/ostris/docs/specification/CORE_WIRE_AND_DECISION_SEMANTICS_V0_1.md` §5 y `JournalCommitService.validatePurpose()`.
- **Observed:** STIR-doc afirma que ningún documento local confirma que `SETTLEMENT` sea propósito real ni sus referencias. La especificación osTRIS vendorizada enumera `SETTLEMENT` y exige `caseId`, `agreementId` o `settlesTransactionId`; el código aplica esa regla. La aplicación de esa semántica a una comisión FFM específica sigue necesitando decisión, pero la existencia del propósito no es un gap.
- **Expected:** separar norma osTRIS confirmada de semántica de fee de distribución aún no aprobada.
- **Impact:** induce a auditores e implementadores a tratar un contrato confirmado como desconocido; puede desviar pruebas y decisiones.
- **Evidence:** ambos documentos y `JournalCommitService` en el pin osTRIS de `upstream.lock.json`.
- **Root cause:** documentación STIR no contrastada con el vendor pin real.
- **Recommended remediation:** corregir texto documental después de conservar este finding; no añadir lógica de fee a STIR.
- **Status:** FOUND → FIXED en rama documental `codex/full-system-audit` → REVALIDATED por comparación con la especificación y `JournalCommitService` del pin actual; aún no integrado en `main`.
- **Regression test:** revisión cruzada de contrato/version pin durante el cambio documental. El texto corregido distingue el propósito osTRIS confirmado del SPEC GAP de comisión de plataforma.

### AUD-006 — HIGH — `idax_admin` puede fabricar un FINAL de Market Integrity

- **Area:** privilegios PostgreSQL / elegibilidad de evidencia.
- **Invariant:** SIGNAL ≠ FINDING; sólo una decisión autorizada y trazable puede excluir observaciones por `FINAL`.
- **Reproduction:** rama `codex/full-system-audit-repro`, commit `b058532`, test `ConsentRetentionPostgresTest#auditReproductionAdminDbRoleCanForgeFinalIntegrityFinding`: sobre PostgreSQL 17.11/V1–V16, crear observación válida, `SET LOCAL ROLE idax_admin`, insertar directamente `market_integrity_case` y `market_integrity_case_event(status='FINAL')`, volver a `idax_app` y calcular manifest. Comando Maven dirigido: PASS.
- **Observed:** la observación aparece como `FINAL_INTEGRITY_FINDING` sin señal/revisión/decisión a través de `MarketIntegrityService`, y sin protección frente al rol SQL administrativo.
- **Expected:** el rol de administración de plataforma no puede crear un finding ni alterar la elegibilidad de mercado; sólo el flujo de servicio autorizado debe escribir esos eventos.
- **Impact:** quien pueda ejecutar SQL como `idax_admin` puede excluir evidencia y fabricar una acusación formal. No se ha demostrado un endpoint HTTP SuperAdmin que permita SQL arbitrario.
- **Evidence:** V6 `GRANT SELECT,INSERT` sobre ambas tablas al rol `idax_admin`, test `b058532`, `ReferenceService.snapshot()` que consume el último evento FINAL.
- **Root cause:** privilegios INSERT administrativos demasiado amplios en tablas de decisión comunitaria.
- **Recommended remediation:** auditar todos los grants comunitarios del rol; revocar mutación administrativa innecesaria en una migración aditiva y conservar acceso de lectura necesario. Revalidar clean install y upgrade, además de una prueba SQL negativa.
- **Status:** FOUND → FIXED (`57cf4d5`, rama backend `codex/audit-high-gate`) → REVALIDATED en PostgreSQL 17.11 local y stack Docker aislado; integrado en `stir-backend/main` (`ea48c08`), VM DEV pendiente.
- **Regression test:** `SET ROLE idax_admin` falla al insertar case/event; el E2E de Community Value Governance completa SIGNAL/UNDER_REVIEW/FINAL/DISMISSED por autoridad legítima.

**Post-fix gate (sin alterar el finding PRE-FIX):** la misma V17 retira INSERT/UPDATE/DELETE administrativos sobre `market_integrity_case` y `market_integrity_case_event`, Reference, Ordinary Governance, Consent/Retention y demás tablas STIR. En el PostgreSQL desplegado localmente, el rol real `idax_admin` recibió `permission denied for table market_integrity_case_event`; no conserva DML en ninguna tabla STIR. El E2E HTTP de Market Integrity completó SIGNAL→UNDER_REVIEW→FINAL y SIGNAL→UNDER_REVIEW→DISMISSED mediante un decisor distinto del originador, con SuperAdmin excluido y aislamiento tenant. Esta prueba no atribuye a PostgreSQL la verificación criptográfica de las decisiones que ejecuta `idax_app`; la credencial privada de runtime sigue siendo un límite de confianza operativo.

### AUD-007 — SPEC GAP (impacto HIGH para un piloto con Seven Keys reales) — reemplazo de controller imposible

- **Area:** recuperación constitucional.
- **Invariant:** pérdida/incapacidad permanente de un controller no debe quedar disfrazada de recuperación completa; Guardian no debe adquirir poder de octava llave.
- **Reproduction:** `SevenKeysService.activate()` devuelve incondicionalmente `FINAL_RESOLUTION_VERIFICATION_UNAVAILABLE` para `REPLACE_CONTROLLER`; `CREDENTIAL_RECOVERY.md` confirma que no hay prueba autenticada transferible de resolución FINAL de continuidad desde osTRIS.
- **Observed:** la rotación del credential de un controller que conserva identidad está implementada; sustituir al controller no puede activarse aun con seis firmas, Guardian y nueva prueba de posesión.
- **Expected:** contrato normativo de verificación FINAL de resolución/continuidad con autoridad, subject, scope, finality, digest y sequence, sin entregar decisión al Guardian ni relajar 7-of-7.
- **Impact:** incapacidad o retirada permanente de un controller puede congelar indefinidamente el control constitucional; un piloto con custodia distribuida real necesita aceptar explícitamente este riesgo o posponerlo. No hay fallback 6-of-7 legítimo.
- **Evidence:** `SevenKeysService.java` rama `REPLACE_CONTROLLER`, `CREDENTIAL_RECOVERY.md` § recovery.
- **Root cause:** falta contrato de verificación entre osTRIS/STIR, no un bug de UI.
- **Recommended remediation:** resolver el SPEC GAP entre autoridades de protocolo, con vectores normativos y pruebas; no implementar durante esta auditoría.
- **Status:** FOUND; abierto.
- **Regression test:** resolución FINAL válida activa sólo el reemplazo exacto; casos no-finales, de otro tenant/community/subject o apelados rechazan.

### AUD-003 — PRODUCT DECISION — alta cross-device en rotación/reemplazo

- **Area:** custodia Seven Keys / WebAuthn.
- **Invariant:** cada titular debe poder demostrar posesión desde su propio dispositivo sin trasladar claves ni convertir al operador de ceremonia en custodio.
- **Reproduction:** `WEBAUTHN_HARDWARE_CUSTODY.md` y `CredentialRegistrar` documentan que `ROTATE_CREDENTIAL`/`REPLACE_CONTROLLER` sólo aceptan alta de credential desde el dispositivo de quien gestiona la propuesta.
- **Observed:** una futura credential ya obtenida en otro dispositivo no puede aportarse a ese formulario; además `REPLACE_CONTROLLER` termina en `FINAL_RESOLUTION_VERIFICATION_UNAVAILABLE` en `SevenKeysService.activate()`.
- **Expected:** ceremonia distribuida viable y verificada, sin pegado de clave pública como supuesto de hardware custody.
- **Impact:** puede bloquear recuperación/rotación real con custodios físicamente separados. Pendiente ensayo de ceremonia en VM para graduar severidad.
- **Evidence:** `WEBAUTHN_HARDWARE_CUSTODY.md`, `SevenKeysService.activate()`.
- **Root cause:** no existe enrollment pendiente cross-device ligado a propuesta/seat/controller/community/action.
- **Recommended remediation:** decisión de producto sobre enrollment pendiente con challenge y proof of possession en dispositivo titular; no implementar durante auditoría sin autorización.
- **Status:** FOUND; sin fix.
- **Regression test:** ceremonia distribuida de rotación/reemplazo con dos dispositivos y rechazo de replay/mixup.

### AUD-008 — MEDIUM — COSE key malformada escapa del rechazo WebAuthn controlado

- **Area:** WebAuthn / parsing de claves públicas COSE.
- **Invariant:** una entrada WebAuthn malformada debe rechazarse como error de cliente sin escapar como fallo interno del servicio; nunca debe aceptar firma ni consumir estado válido.
- **Reproduction:** rama aislada `codex/full-system-audit-repro`, `WebAuthnCryptoTest#auditCoseKeyMissingAlgorithmFailsAsClientError`: construir un mapa CBOR válido con `kty=EC2` pero sin campo obligatorio `alg`; llamar `WebAuthnCrypto.publicKeyFromCose()`. `mvn -q -Dtest=WebAuthnCryptoTest#auditCoseKeyMissingAlgorithmFailsAsClientError test` falla: esperaba `IllegalArgumentException` y recibió `NullPointerException` en `WebAuthnCrypto.java:135`. Un test adicional con CBOR truncado sí obtuvo rechazo controlado.
- **Observed:** `map.get(3L)` devuelve null y se invoca `.longValue()` sin validar presencia/tipo. `WebAuthnCredentialService` y `SevenKeysService` convierten sólo `IllegalArgumentException` en HTTP 400; el NPE escapa de esa frontera. Todavía no se ha medido el código HTTP real en VM.
- **Expected:** rechazo tipado (`IllegalArgumentException` en el parser, HTTP 400 en la ruta) para mapa incompleto, tipo incorrecto, algoritmo no soportado y claves malformadas.
- **Impact:** un cliente puede provocar error interno en registro/assertion WebAuthn; el alcance de disponibilidad o filtrado de stack trace requiere ensayo HTTP. No hay evidencia de bypass criptográfico ni de aceptación de credential malformada.
- **Evidence:** fallo Surefire dirigido con excepción y línea; `WebAuthnCrypto.publicKeyFromCose()` y manejadores de servicio.
- **Root cause:** acceso no validado a campos COSE obligatorios, fuera del bloque que traduce errores de parsing.
- **Recommended remediation:** validar presencia/tipo/longitud de campos COSE antes de construir clave pública, convertir todos los errores de entrada al mismo rechazo tipado y añadir vectores negativos. No ampliar el conjunto de algoritmos ni cambiar la política de attestation.
- **Status:** FOUND → FIXED en rama backend separada `codex/audit-webauthn-cose` commit `8e4003a` → REVALIDATED localmente. No integrado en `main` ni desplegado en VM.
- **Post-fix validation:** los vectores negativos de COSE incompleta, tipo incorrecto, coordenada ausente, CBOR truncado y `authData` de tipo incorrecto dan `IllegalArgumentException`; `mvn -q verify` pasó con 220 tests, 0 fallos/errores/omitidos, y V1–V16 desde PostgreSQL limpio. Después se añadió una prueba de servicio dirigida: registro malformado devuelve 400 y no persiste credential, PASS. El test pre-fix en `codex/full-system-audit-repro` commit `14146db` continúa fallando en `main` y preserva la evidencia.
- **Regression test:** vectores dirigidos y servicio con PostgreSQL pasan; aún falta HTTP real en VM.

### AUD-009 — MEDIUM — pin de object storage anterior a releases de seguridad del fork

- **Area:** MinIO/PGSTY SILO, supply chain y operabilidad.
- **Invariant:** una imagen pinneada debe ser reproducible **y** mantener una ruta de mantenimiento evaluada; para piloto, fotos/objetos deben sobrevivir a fallo y restauración.
- **Reproduction:** `stir-main/compose.yml` fija `pgsty/minio@sha256:b6bfe7239bfc83fb90d31612d9704d86039dd714f7904b3f1ad68f211e602372`. El [issue del mantenedor sobre esa imagen](https://github.com/pgsty/silo/issues/55) la identifica como release 2026-08-04 (manifest index). El [release 2026-09-16 del mismo proyecto](https://github.com/pgsty/silo/releases/tag/RELEASE.2026-09-16T00-00-00Z) publica fixes posteriores de autenticación/policies y durabilidad IAM, e indica que la línea mantenida se llama `pgsty/silo`.
- **Observed:** pin por digest reproducible, pero de una línea/fecha anterior a fixes publicados; el comentario de Compose aún presenta `pgsty/minio` como fork mantenido. `minio` sólo está en la red interna `data` en Compose, sin puerto host, lo que reduce exposición directa. No se ha confirmado aún el digest realmente cargado en VM DEV ni el estado de su volumen/credenciales.
- **Expected:** inventario de digest desplegado, matriz de fixes aplicables al uso STIR, ensayo de upgrade/rollback en copia aislada y decisión de pin soportado antes de piloto.
- **Impact:** se podrían arrastrar defectos de seguridad/corrección ya resueltos por el mantenedor; migrar a ciegas también podría afectar formato IAM o recuperación. No se afirma explotación concreta contra STIR.
- **Evidence:** Compose local y documentación/release primaria del mantenedor enlazados arriba.
- **Root cause:** cambio rápido de distribución del fork tras el pin inicial; falta revisión periódica de actualización.
- **Recommended remediation:** evaluar release mantenido más reciente en copia DEV con backup/restore y S3/photo regression; fijar un digest verificado y documentar compatibilidad. No sustituir ahora el object storage ni cambiar el pin de la VM sin checkpoint.
- **Status:** FOUND como riesgo supply-chain del `main` local; VM pendiente.
- **Regression test:** comprobar digest del contenedor, health, upload/read SHA-256, restart, backup/restore y permisos en la versión propuesta.

### AUD-010 — HIGH — fallo transitorio de osTRIS queda registrado como rechazo económico definitivo

- **Area:** STIR→osTRIS, ejecución económica y recuperación de fallos.
- **Invariant:** `REJECTED` debe significar rechazo económico conocido por osTRIS; un 5xx/resultado desconocido no demuestra ni fallo de firmas ni rechazo de política, y nunca debe convertirse en estado terminal local sin reconciliación.
- **Reproduction:** rama de reproducción backend `codex/full-system-audit-repro` commits `7f122c3` y `f7abb95`. `TradeServiceTest#auditReproductionTransientOstrisFailureRecordsPermanentRejection`: `ostris.commit(transactionId)` lanza `StirOstrisException(503,"UPSTREAM_UNAVAILABLE",...)`; `TradeService.commit()` devuelve 422 y llama `TradeRejectionRecorder.reject()`. `auditReproductionCommitBeforeBothSignaturesRecordsPermanentRejection`: osTRIS devuelve el 422 real `CONTROL_POLICY_NOT_SATISFIED` antes de todas las firmas y STIR también registra rechazo terminal. Ambos Maven dirigidos: PASS. El código osTRIS fijado emite ese error cuando falta el threshold y no marca la propuesta remota REJECTED.
- **Observed:** el recorder `REQUIRES_NEW` persiste `trade.executionState='REJECTED'` y `agreement.economicPhase='REJECTED'` aunque el estado remoto sea desconocido o aún falten firmas. Si el commit remoto no ocurrió y sigue PROPOSED, llamadas posteriores a `commit()` se rechazan por el estado local; `activate()` devuelve el Trade existente y no crea vía de recuperación. Si ocurrió, `sync()` puede reconciliarlo a COMMITTED, pero requiere intervención y el rechazo/notifications falsos ya se emitieron.
- **Expected:** distinguir rechazo normativo terminal de fallo transitorio/indeterminado; consultar osTRIS y reintentar de modo idempotente cuando sea seguro, sin fabricar ejecución ni rechazo. `ProposalAuthorizationService.commit()` del pin osTRIS devuelve el recibo existente si la transacción ya está committed.
- **Impact:** pérdida de disponibilidad de ejecución económica y estado/auditoría engañosos ante fallo de red o upstream. No demuestra doble cargo ni falsificación de commit, pero puede dejar Agreements reales sin cierre normal.
- **Evidence:** test `7f122c3`, `TradeService.commit()`, `TradeRejectionRecorder.reject()` y `ProposalAuthorizationService.commit()` vendorizado en `upstream.lock.json`.
- **Root cause:** catch indiscriminado de todo `StirOstrisException` como rechazo 422 definitivo.
- **Recommended remediation:** separar errores definitivos de transporte/5xx/429/estado desconocido; preservar `AWAITING_SIGNATURES`/estado reconciliable ante incertidumbre, conservar código upstream y permitir `sync`/reintento idempotente. Probar 422 normativo, 503 antes/después de commit y solicitudes concurrentes en VM antes de integrarlo.
- **Status:** FOUND → FIXED (`ea48c08`, rama backend `codex/audit-high-gate`; E2E `ee4ef11` en stir-main) → REVALIDATED en stack local aislado y 224 tests backend; integrado en ambos `main` locales, VM DEV pendiente.
- **Post-fix validation:** `mvn -q verify`: 224 tests, 0 fallos; los tests económicos cubren matriz 400/401/403/404/408/409/422/429/503, timeout, firmas incompletas, retry, reconciliación de falso REJECTED legado y éxito posterior. El E2E final con fault proxy y osTRIS real pasó dos veces (antes y después del último mapeo HTTP); una respuesta perdida tras commit no produjo doble journal.
- **Regression test:** el script `scripts/audit_economic_commit_e2e.py` y el `economic_exchange_e2e.py` actualizado comprueban el estado pendiente después de 422/503 y la cardinalidad de journal; queda la futura auditoría de concurrencia/VM DEV fuera de este gate.

**Post-fix gate:** `codex/audit-high-gate` elimina `TradeRejectionRecorder` y nunca infiere `REJECTED` de una excepción de `commit`. El osTRIS fijado sólo persiste PROPOSED/COMMITTED; un error 422 (incluido límite de crédito) no persiste rechazo remoto. STIR conserva la ejecución pendiente ante 400/401/403/404/408/409/422/429/5xx y timeout, responde con categoría HTTP distinguida, y permite reintento con el mismo `transactionId`; un viejo REJECTED local se reconcilia desde osTRIS antes de reintentar. `TradeServiceTest` cubre la matriz de errores y timeout. E2E Docker aislado contra STIR+osTRIS reales con proxy de fallos: firmas incompletas→409 sin rechazo; 503 antes del procesamiento→pendiente; timeout antes del procesamiento→pendiente; reintento→COMMITTED; pérdida de respuesta después del commit remoto→reconciliación; retry duplicado→una sola fila `ostris.journal_transaction`; 422 de crédito→sin journal ni movimiento de balances, ejecución pendiente. Los estados de Agreement aceptado y ejecución permanecieron separados. No se atribuye semántica terminal a un código 4xx/5xx por conveniencia. Una futura respuesta remota distinta de PROPOSED/COMMITTED requerirá contrato explícito (SPEC GAP de evolución, no estado inventado en STIR).

## Cobertura pendiente

Las fases activas de tenant A/B, SuperAdmin, votos, Seven Keys, WebAuthn virtual, independencia, fuentes/lineage, consent, referencias, extensión, osTRIS, concurrencia, actualización, reinicios, backup/restore en DB separado, MinIO y navegador HTTPS VM **no están aún ejecutadas**. `PILOT_READINESS.md` reservará el dictamen hasta tener evidencia suficiente; ningún PASS local reemplaza estos controles.

`stir-main/scripts/backup_restore_e2e.py` destruye expresamente los dos volúmenes `stir-dev` y no es apto para la VM DEV compartida. `AUDIT_RUNBOOK.md` define el restore aislado requerido. `backup.py` produce primero un `pg_dump` y después un tar del volumen de objetos; bajo escrituras concurrentes esos dos artefactos no constituyen por sí solos un snapshot atómico. Falta ensayo real de consistencia y recuperación antes de puntuar recoverability.
