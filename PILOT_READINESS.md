# Pilot readiness — STIR/osTRIS/IDAX

**Dictamen del baseline `main` auditado: NOT PILOT READY.** Hay tres findings HIGH abiertos en `main`: el rol SQL administrativo puede insertar constitución no firmada (`AUD-004`), fabricar un FINAL de Market Integrity (`AUD-006`) y STIR vuelve terminal un commit osTRIS pendiente ante 503 o firmas incompletas (`AUD-010`). Los dos primeros vulneran el límite de autoridad comunitaria/constitucional en la base y el tercero la integridad del estado económico. Según los criterios de esta auditoría, cualquiera impide PILOT READY mientras siga abierto.

**Alcance del dictamen histórico:** código y migraciones de los cinco `main` locales antes del HIGH remediation gate. Tras cerrar el gate, el acceso SSH a DEV permitió confirmar los mismos cinco HEAD, el despliegue V16 y el rol `idax_admin` todavía con permisos PRE-FIX. Se creó un checkpoint de PostgreSQL y objetos, legible pero aún sin prueba de restore. `stir-main` contiene `.venv/` no versionado cuya procedencia está pendiente; por ello no se han modificado repositorios ni actualizado el stack VM. No se atribuye a la VM el estado V17 ni las correcciones locales. Tenant A/B, restore y navegador autenticado en VM siguen inconclusos.

**HIGH remediation gate: reabierto tras prueba adversarial ampliada.** `AUD-004`, `AUD-006` y `AUD-010` están `FOUND → FIXED → REVALIDATED` para sus reproducciones originales e integrados en `stir-backend/main` (`ea48c08`) y `stir-main/main` (`ee4ef11`). PostgreSQL 17.11: V1→V17 limpio, V16→V17 con datos previos, DML de `idax_admin` denegado en todas las tablas STIR. Backend: 224 tests; E2E económico, Ordinary Governance, Market Integrity, Seven Keys, WebAuthn y Consent/Retention pasan localmente. Pero sobre una copia restaurada de DEV, `idax_app` todavía pudo insertar directamente una constitución no autorizada y un FINAL (`AUD-012`, HIGH abierto). No se desplegará V17 al stack DEV activo ni se declarará cerrado el límite de autoridad hasta resolver si ese rol es una credencial privilegiada fuera del alcance o restringir técnicamente sus escrituras gobernadas. Las pruebas usaron `ROLLBACK`; DEV activo sigue en V16.

**Dictamen operativo actual: NOT PILOT READY.** `AUD-012` mantiene abierto el límite de autoridad según la formulación estricta del usuario; el despliegue DEV activo aún conserva V16 y los HIGH originales. Este dictamen se revisará sólo tras resolver el límite de `idax_app`, actualizar DEV y completar los ensayos de aislamiento, navegador y recoverability pendientes.

El [estudio de auditoría resistente a manipulación](GOVERNED_STATE_AUDIT_ARCHITECTURE.md) está
integrado en `main` (2026-09-30, tras `PHASE 1 INTERNAL IMPLEMENTATION GATE: ACCEPTED` - ver
`VALIDATION_GOVERNED_STATE_AUDIT_MVP.md`), pero sigue sin ser un control desplegado en ningún
entorno vivo (DEV/PROD). Su hash chain/verificador/anchor no justifican por sí solos cambiar
`AUD-012` ni el dictamen: una historia MI/Ordinary fabricada pero autoconsistente y el consumo antes
de verificación siguen siendo riesgos abiertos.

Su **Fase 1 ya está implementada, validada y ahora integrada en `main`**
([VALIDATION_GOVERNED_STATE_AUDIT_MVP.md](VALIDATION_GOVERNED_STATE_AUDIT_MVP.md)): migración V18/
V19, trigger de auditoría en 40/47 tablas reales, verificador Java separado con `/security-status`
(alerta durable canónica, DB-backed), ambos dos ataques directos de `AUD-012` detectados de forma
reproducible contra roles PostgreSQL reales. **No cambia el dictamen NOT PILOT READY**: ningún
dominio alcanza `PASS_CRYPTO`, no hay consumption gate, no hay anclaje externo implementado, y
aunque los commits ya están integrados y pusheados a `main` en los cuatro repos, **no se ha
desplegado en el stack DEV activo (`stir-dev`)**. Codex aceptó la Fase 1 tras cinco rondas de
reauditoría independiente (`FASE 1 ACEPTABLE`, ver los cinco `*_REVALIDATION_GOVERNED_STATE_AUDIT_
PHASE1.md`); la integración completa del sistema (stack aislado con IDAX Core/Shell, osTRIS, IDAX
Ledger) sobre la VM DEV está pendiente y se documentará por separado en
`INTEGRATION_GOVERNED_STATE_AUDIT_PHASE1.md` cuando se ejecute.

| Control | Evidencia actual | Estado |
|---|---|---|
| Baseline cinco repos STIR PC | HEAD/branch/status/remote registrados en `FULL_SYSTEM_AUDIT.md` | VERIFICADO en PC |
| Baseline VM y pins reales | SSH puerto 12522, cinco HEAD coincidentes, STIR V16, PostgreSQL 17.11, contenedores, imágenes, volúmenes y runtimes registrados; `stir-main/.venv/` no versionado | PARCIAL; checkout VM sin modificar |
| Clean PostgreSQL V1–V16 + backend | Testcontainers PostgreSQL 17.11; 216 tests PASS | VERIFICADO localmente |
| Autorización de rol DB comunitario | V17: cero tablas STIR con DML `idax_admin`; en copia DEV, `idax_app` todavía insertó constitución y FINAL sin autoridad (`AUD-012`) | HIGH ABIERTO; límite de confianza pendiente |
| Commit económico ante osTRIS | PRE-FIX: 503/firmas incompletas causan `REJECTED`; POST-FIX: pendientes/retry y journal único tras fallos controlados | REVALIDATED local/main; VM pendiente |
| Tenant A/B y SuperAdmin HTTP/DB VM | fixtures y credenciales pendientes | NOT RUN |
| osTRIS económico, no-FX, extensión externa | E2E económico real local y auditoría parcial de contrato; no-FX estático | PARCIAL; VM/extensión pendientes |
| WebAuthn en VM, virtual y hardware manual | HTTP E2E local pasa replay/tenant/comunidad/rotación/6-of-7; navegador/hardware VM pendiente | PARCIAL |
| WebAuthn malformado | `AUD-008` reproducido pre-fix; fix `8e4003a` validado localmente | FIX LOCAL; VM/main pendientes |
| Object storage | digest reproducible pero anterior a releases con fixes (`AUD-009`) | REVISIÓN/RESTORE pendientes |
| Upgrade desde versión anterior con datos | Flyway 11.12.0 aplicó V16→V17 en copia del dump DEV: 28 constituciones y 26 credentials conservadas, cero DML `idax_admin` | PASS Flyway aislado; boot/stack VM pendientes |
| Reinicios y durabilidad VM | pendiente | NOT RUN |
| Backup/restore a entorno aislado | checkpoint DEV: primer restore falló sin roles globales; segundo `pg_restore --exit-on-error` pasó tras recrearlos, 48 tablas STIR/216 políticas RLS/26 credentials coinciden; MinIO restaurado da health 200; faltan fixtures no vacíos y boot de STIR contra la copia | PARCIAL; recoverability no demostrada |
| Navegador HTTPS real con sesión | login público cargó; sesión no disponible | INCOMPLETO |

## Condiciones para una nueva evaluación

1. Resolver `AUD-012` y aclarar procedencia de `stir-main/.venv/` antes de actualizar DEV; luego repetir grants/RLS/E2E en la VM.
2. Registrar/corregir/revalidar cada HIGH técnico en rama específica y en upgrade real, sin alterar 7-of-7 ni introducir poderes Guardian.
3. Completar intentos adversariales `AUDIT-A`/`AUDIT-B`, WebAuthn, osTRIS, extensión, concurrencia e historial en VM.
4. Demostrar reinicios y restore real en DB/objetos separados; verificar histórico, referencias, Agreements, credenciales públicas y fotos.
5. Resolver o aceptar formalmente los `SPEC GAP`/`PRODUCT DECISION` según alcance de piloto. La fiscal valuation transaccional, si aparece, nunca será paridad osTRIS/fiat ni ingreso imponible/cuota fiscal del vendedor.

No hay score ni porcentaje. La tabla distingue pruebas realizadas de las pendientes.

## Respuestas a las preguntas de auditoría

| Pregunta | Respuesta basada en evidencia actual |
|---|---|
| 1. ¿Puede A afectar B? | **INCONCLUSO:** tests locales parciales de RLS; sin fixtures AUDIT-A/B en VM. |
| 2. ¿Puede Platform SuperAdmin gobernar? | **Baseline FAIL DB**; V17 local quita todo DML STIR a `idax_admin`, dos INSERT reales dan `permission denied`, y HTTP E2E excluye SuperAdmin. VM/main pendientes. |
| 3. ¿Guardian octava llave? | Tests locales rechazan sustitución de firma; ceremonia VM y concurrencia pendientes. |
| 4. ¿6-of-7 constitucional? | Tests locales rechazan amendment 6-of-7; VM pendiente. La excepción documentada 5-of-7 sólo remueve Guardian. |
| 5–6. ¿Replay/cross-community WebAuthn? | Tests locales cubren ambos rechazos; browser VM pendiente. COSE malformada reveló `AUD-008`. |
| 7. ¿Withdrawal borra evidencia? | Tests locales preservan snapshot; `AUD-002` revela hold potencialmente indefinido tras DISMISSED. VM pendiente. |
| 8–10. ¿Cuentas, lineage y seed inflan evidencia? | Cobertura unitaria existente, pero ataque activo con cuentas/relists/seed en VM pendiente. |
| 11. ¿Extensión salta invariantes? | Baseline `idax_admin` podía hacerlo; V17 local cierra esa vía. Contrato público externo y privilegios VM aún sin prueba activa. |
| 12. ¿Se modifica external contract aceptado? | Snapshot/digest en código; replay y mutación adversarial VM pendientes. |
| 13. ¿STIR fuerza osTRIS? | Cliente sólo `EXCHANGE` con firmas de cuentas; E2E local confirma journal único y ningún commit por rechazo/error. VM pendiente. |
| 14. ¿Paridad fiat/osTRIS? | Búsqueda estática acotada sin paridad; runtime/contratos externos pendientes. |
| 15. ¿Migraciones de DB limpia? | PASS local Testcontainers PostgreSQL 17.11 para STIR V1–V17 y stack Docker local; VM pendiente. |
| 16–17. ¿Reinicio/restore? | NOT RUN en VM. Restore debe usar DB y volúmenes separados. |
| 18. ¿HTTPS/browser VM? | Login público y health GET cargan; sesión y recorridos E2E NOT RUN. |
| 19. ¿Gaps bloqueantes? | Los tres HIGH fueron corregidos/revalidados e integrados localmente, no desplegados/medidos en VM; `AUD-007` recovery controller conserva impacto HIGH para piloto. |
| 20. ¿Riesgos aceptables? | Aún no aceptados formalmente: `AUD-001`, `AUD-002`, `AUD-003`, `AUD-009` y cobertura pendiente. |
