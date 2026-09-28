# FULL SYSTEM + DEV VM audit runbook

Estado: procedimiento de auditoría, no constancia de ejecución. Usar sólo VM DEV y tenants sintéticos `AUDIT-*`. Registrar cada resultado en `FULL_SYSTEM_AUDIT.md` antes de cualquier fix.

## 1. Congelar baseline antes de cualquier escritura

1. Confirmar host, usuario y ruta real de DEV; no reutilizar una IP histórica sin verificar. Confirmar que el destino no es producción.
2. En cada repositorio STIR y cada dependencia real: ruta, `git status --porcelain=v1`, rama, HEAD y remote. Detener modificaciones si otro agente/sesión ensució un checkout. Comparar el `upstream.lock.json` con los HEAD vendorizados/desplegados.
3. Registrar versión de `docker`, Java, Node, Python, PostgreSQL y navegador; `docker compose ps`, IDs/digests de imágenes, volúmenes y montajes, endpoints y versiones públicas. Redactar secretos antes de guardar salidas. Consultar `flyway_schema_history` por esquema y catálogo de grants/RLS/FORCE RLS.
4. Hacer checkpoint seguro de DEV. Conservar nombre, hora UTC, tamaño y SHA-256 de dump PostgreSQL y archivo de objetos; comprobar que son legibles. No usar `--keep` que pueda podar backups anteriores.

## 2. Dataset y registro

Crear `AUDIT-A` y `AUDIT-B` con cuentas no reales, permisos mínimos y comunidades osTRIS separadas. Registrar IDs, versiones y relaciones en un manifiesto local privado de auditoría; no guardar credenciales en Git. Antes de cada intento adversarial, registrar operación, identidad, tenant, método/ruta o SQL, IDs de request/correlation, respuesta y estado DB anterior. Después, consultar estado DB y audit trail. Una denegación HTTP no basta si el estado cambió.

Ejecutar la matriz de ataques del encargo por áreas. Para concurrencia, enviar dos solicitudes con la misma intención/idempotency key desde barrera simultánea, guardar ambas respuestas y consultar cardinalidad de rows/eventos/transactions. Para inmutabilidad histórica, guardar digest de snapshots y volver a leerlo tras cada cambio posterior. Mantener una tabla de controles `PASS`, `FAIL`, `NOT RUN`, `INCONCLUSIVE`; nunca convertir `NOT RUN` en `PASS`.

## 3. Recorrido HTTPS real

Desde navegador integrado y, cuando aplique, authenticator virtual: `https://dev.stir.es` → reverse proxy → frontend → API → DB/osTRIS. Registrar origen, certificado, cookies (flags, SameSite), CORS, CSP, sesiones, upload y descarga de fotos. Recorrer alta/login, OFFER/WANTED, negociación/Agreement, exchange, reference, ordinary governance, consent, integrity, Seven Keys y WebAuthn. Usar sólo fixtures `AUDIT-*`. El authenticator virtual prueba contratos de navegador, no custodia física ni propiedades de hardware.

Checklist manual con authenticator físico en navegador externo: registrar RP ID y origin; inscripción de cada seat/Guardian desde dispositivo titular; confirmación UV; firma 7-of-7; rechazo 6-of-7; replay/rotación/revocación; ceremonia cross-device para rotar; pérdida de controller y comportamiento fail-closed. No completar una ceremonia constitucional real fuera de AUDIT-*.

## 4. Backup/restore no destructivo para DEV compartido

`stir-main/scripts/backup_restore_e2e.py` **elimina** `stir-dev_postgres_data` y `stir-dev_minio_data`; no ejecutarlo en la VM DEV compartida. El test de esta auditoría debe:

1. Crear marker `AUDIT-*` con listing/foto y, si existen fixtures seguros, Agreement, reference, governance e historial WebAuthn público.
2. Tomar backup de PostgreSQL y objetos, documentando la ventana temporal entre ambos. Comprobar SHA-256 y contenido esperado.
   Antes del restore en un cluster PostgreSQL vacío, recrear en el cluster **aislado** los roles globales y memberships del origen que referencian las políticas RLS; `pg_dump` de una base no los incluye. Usar credenciales sintéticas nuevas y no copiar passwords DEV. Ejecutar `pg_restore --exit-on-error` para que una política fallida no pase inadvertida.
3. Levantar una composición/DB/volúmenes/puertos **separados** con credenciales de prueba propias y sin rutas públicas ni callbacks de producción. Restaurar allí el dump y los objetos. Nunca ejecutar `restore.py` sobre el proyecto o volumen original.
4. Arrancar backend/servicios contra la copia, verificar migraciones, login de auditoría, bytes de la foto, Agreements, referencias, governance, historial y material público WebAuthn. Registrar errores y checksums.
5. Limpiar exclusivamente la composición aislada tras verificar su nombre/proyecto/paths; conservar el backup original y el informe.

Un dump SQL y un tar de objetos hechos secuencialmente no son automáticamente un snapshot consistente bajo escrituras concurrentes. Probar una carga/upload concurrente o definir una ventana de quiesce antes de declarar recoverability de piloto.

## 5. Restart y upgrade

Con checkpoint y fixtures: reiniciar frontend, backend, stack y, sólo si es seguro en DEV, PostgreSQL. Releer estado e historial tras cada reinicio. Para upgrade, partir de una versión anterior fijada por commit/imagen y migrar **la copia aislada** con datos representativos; registrar desde→hasta y aplicar contratos de compatibilidad. No inferir compatibilidad universal de una única ruta.

## 6. Findings y dictamen

Cada defecto: `ID`, `Severity`, `Area`, `Invariant`, `Reproduction`, `Observed`, `Expected`, `Impact`, `Evidence`, `Root cause`, `Recommended remediation`, `Status`, `Regression test`. Registrar `FOUND` antes del fix; en ramas específicas, `FIXED` y `REVALIDATED` después de repetir el caso y suite apropiada. Separar `SPEC GAP` y `PRODUCT DECISION` de bugs.

El dictamen en `PILOT_READINESS.md` debe seguir estrictamente los criterios del encargo. Si faltan VM, E2E, restart o restore, declararlos `NOT RUN`/`INCONCLUSIVE`; ninguna suite local sustituye esas pruebas.

## 7. Repetir el HIGH remediation gate en un stack local aislado

Después de integrar la rama de corrección, usar `stir-main/compose.yml` con `stir-main/compose.audit-commit.yml` y un proyecto Docker distinto de `stir-dev` (por ejemplo `-p stir-audit-high`). Elegir `STIR_HTTP_PORT=18089` y `STIR_API_PORT=18096`, comprobar que están libres y levantar con `docker compose -f compose.yml -f compose.audit-commit.yml -p stir-audit-high up -d --build`. El overlay enruta únicamente ese stack a un proxy de fallos osTRIS de un solo uso en `127.0.0.1:18095`.

Con `COMPOSE_PROJECT_NAME=stir-audit-high` y `STIR_TEST_URL=http://localhost:18089`, ejecutar `python scripts/audit_economic_commit_e2e.py`. La prueba se niega a apuntar a un host no local, crea un tenant `AUDIT-*`, comprueba firmas incompletas, 503, timeout, pérdida de respuesta tras commit, retry y un único journal osTRIS, más rechazo de crédito sin movimiento económico. Ejecutar también `mvn verify` en `stir-backend`; `StirAdminAuthorityPostgresTest` comprueba DML de `idax_admin` sobre todo el esquema STIR y `StirAdminAuthorityUpgradePostgresTest` valida V16→V17 con historia existente. Los scripts existentes `economic_exchange_e2e.py`, `ordinary_governance_e2e.py`, `community_value_governance_e2e.py`, `market_integrity_e2e.py`, `consent_retention_e2e.py` y `webauthn_hardware_custody_e2e.py` son la regresión HTTP de servicios.

Para prueba manual de DB en el proyecto aislado, consultar `has_table_privilege('idax_admin','stir.market_constitution','INSERT')` y el equivalente de `market_integrity_case_event`, más `has_any_column_privilege`; debe devolver falso. Un `INSERT` directo tras `SET LOCAL ROLE idax_admin` y `SET LOCAL app.tenant_id` debe fallar por `permission denied`, antes de llegar a constraints o RLS. El owner/Flyway y PostgreSQL superuser son infraestructura de confianza; la prueba no afirma resistencia ante root/superuser ni ante robo de la credencial privada `idax_app`.

Al terminar, detener sólo el proyecto `stir-audit-high` iniciado por la auditoría (`docker compose ... -p stir-audit-high down`, sin `-v`); conservar sus volúmenes para evidencia hasta decidir su archivo seguro.
