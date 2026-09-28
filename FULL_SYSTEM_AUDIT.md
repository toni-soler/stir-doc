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

PC: Docker client/server 27.0.3; Java 25.0.1; Node 20.16.0; Python 3.14.3. La composición local `stir-dev` apareció al primer inventario, con imágenes `stir` `sha256:dd004f56`, Shell `sha256:1c27a408`, osTRIS `sha256:45b503b8`, Ledger `sha256:f13f700a`, y volúmenes `stir-dev_postgres_data` y `stir-dev_minio_data`. Antes de poder consultar sus migraciones los contenedores desaparecieron de Docker Desktop por una operación externa a esta auditoría; los volúmenes permanecieron. No se reinició ni alteró esa composición. Esta discrepancia impide usarla como baseline estable de DEV.

La VM DEV está documentada como `dev.stir.es`, con Nginx Proxy Manager terminando TLS. `https://dev.stir.es/api/stir/instance` respondió HTTP 200 el 2026-09-28 con `stirVersion=0.5.0-rc1`, catálogo 1 y snapshot externo 3; `/actuator/health/readiness` respondió 200 con `X-Correlation-Id`/`X-Request-Id`. La conexión SSH documentada a `stir-admin@192.168.1.125:22` fue rechazada; **no se han verificado aún** HEAD/status de VM, imágenes, migraciones, roles/RLS, volúmenes, versiones de runtime, backup ni configuración privada DEV. Se ha pedido el acceso actual. Ninguna afirmación sobre sincronización PC↔VM se toma por demostrada hasta comparar hashes.

## Evidencia ejecutada sobre `main` sin modificaciones

- `mvn -q verify` en `stir-backend`: 216 tests, 0 fallos/errores/omitidos. Testcontainers PostgreSQL 17.11 aplicó V1–V16 desde esquema `stir` vacío en varias instancias. Esto no prueba la composición entera ni la base real de VM.
- `npm test` en `stir-frontend`: 45 tests, 0 fallos. `npm run build` y `npm run i18n:validate`: PASS, 12 locales y 560 claves.
- `python scripts/audit-public.py` en `stir-main`: PASS para 327 ficheros STIR propios. Es una búsqueda acotada, no un escáner completo de secretos/dependencias.
- HTTPS DEV: certificado aceptado por `curl`, proxy `openresty`, HSTS, CSP, `X-Content-Type-Options`, `X-Frame-Options` presentes. Falta sesión, upload, WebAuthn y recorrido de navegador autenticado.

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
- **Status:** FOUND; reproducido; sin fix.
- **Regression test:** `idax_admin` no puede insertar `market_constitution` ni mutar otros objetos constitucionales, `idax_app` sólo por rutas de servicio firmadas; verificar permisos en base limpia y upgrade.

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
- **Status:** FOUND; reproducido; sin fix.
- **Regression test:** `SET ROLE idax_admin` no puede insertar case/event, y las rutas de servicio con `idax_app` siguen pudiendo registrar signal/review/decision legítimos.

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

## Cobertura pendiente

Las fases activas de tenant A/B, SuperAdmin, votos, Seven Keys, WebAuthn virtual, independencia, fuentes/lineage, consent, referencias, extensión, osTRIS, concurrencia, actualización, reinicios, backup/restore en DB separado, MinIO y navegador HTTPS VM **no están aún ejecutadas**. `PILOT_READINESS.md` reservará el dictamen hasta tener evidencia suficiente; ningún PASS local reemplaza estos controles.
