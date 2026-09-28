# Pilot readiness — STIR/osTRIS/IDAX

**Dictamen del baseline `main` auditado: NOT PILOT READY.** Hay tres findings HIGH abiertos en `main`: el rol SQL administrativo puede insertar constitución no firmada (`AUD-004`), fabricar un FINAL de Market Integrity (`AUD-006`) y STIR vuelve terminal un commit osTRIS pendiente ante 503 o firmas incompletas (`AUD-010`). Los dos primeros vulneran el límite de autoridad comunitaria/constitucional en la base y el tercero la integridad del estado económico. Según los criterios de esta auditoría, cualquiera impide PILOT READY mientras siga abierto.

**Alcance del dictamen histórico:** código y migraciones de los cinco `main` locales antes del HIGH remediation gate, pins vendorizados y sondas públicas limitadas a `https://dev.stir.es`. El SSH a VM DEV quedó disponible el 28-09-2026 y permitió confirmar los mismos cinco HEAD y contenedores principales; `stir-main` tenía `.venv/` no versionado. El usuario congeló la auditoría VM antes de crear fixtures o modificarla. No se atribuye a la VM el estado V17 ni las correcciones locales: sigue en V16 hasta una actualización controlada. Tenant A/B, backup/restore y navegador autenticado en VM siguen inconclusos.

**HIGH remediation gate local:** `AUD-004`, `AUD-006` y `AUD-010` están `FOUND → FIXED → REVALIDATED` en ramas aisladas. PostgreSQL 17.11: V1→V17 limpio, V16→V17 con datos previos, DML de `idax_admin` denegado en todas las tablas STIR, intentos directos de constitución y FINAL rechazados por el rol real. Backend: 224 tests, 0 fallos. E2E Docker aislado con osTRIS real: retry tras 503/timeout, respuesta perdida tras commit, journal único, 422 de crédito sin movimiento y estado pendiente. Ordinary Governance, Market Integrity, Seven Keys, WebAuthn y Consent/Retention HTTP E2E pasan en tenants de auditoría. **Aún no es un nuevo dictamen de piloto**: faltan integración `main`, actualización/validación VM y el resto de fases. El trust boundary de propietario/superuser PostgreSQL y credencial privada `idax_app` está descrito en `FULL_SYSTEM_AUDIT.md`.

| Control | Evidencia actual | Estado |
|---|---|---|
| Baseline cinco repos STIR PC | HEAD/branch/status/remote registrados en `FULL_SYSTEM_AUDIT.md` | VERIFICADO en PC |
| Baseline VM y pins reales | SSH `devstires` y cinco HEAD coincidentes; `stir-main/.venv/` no versionado; contenedores/digests iniciales registrados | PARCIAL; VM pausada por usuario |
| Clean PostgreSQL V1–V16 + backend | Testcontainers PostgreSQL 17.11; 216 tests PASS | VERIFICADO localmente |
| Autorización de rol DB comunitario | PRE-FIX: inserts admitidos; POST-FIX V17: cero tablas STIR con DML `idax_admin`, dos INSERT directos denegados | REVALIDATED local; main/VM pendientes |
| Commit económico ante osTRIS | PRE-FIX: 503/firmas incompletas causan `REJECTED`; POST-FIX: pendientes/retry y journal único tras fallos controlados | REVALIDATED local; main/VM pendientes |
| Tenant A/B y SuperAdmin HTTP/DB VM | fixtures y credenciales pendientes | NOT RUN |
| osTRIS económico, no-FX, extensión externa | E2E económico real local y auditoría parcial de contrato; no-FX estático | PARCIAL; VM/extensión pendientes |
| WebAuthn en VM, virtual y hardware manual | HTTP E2E local pasa replay/tenant/comunidad/rotación/6-of-7; navegador/hardware VM pendiente | PARCIAL |
| WebAuthn malformado | `AUD-008` reproducido pre-fix; fix `8e4003a` validado localmente | FIX LOCAL; VM/main pendientes |
| Object storage | digest reproducible pero anterior a releases con fixes (`AUD-009`) | REVISIÓN/RESTORE pendientes |
| Upgrade desde versión anterior con datos | STIR V16→V17 con constitución histórica conservada en PostgreSQL local | PASS acotado; upgrade VM integral pendiente |
| Reinicios y durabilidad VM | pendiente | NOT RUN |
| Backup/restore a entorno aislado | procedimiento seguro en `AUDIT_RUNBOOK.md`; ejecución pendiente | NOT RUN |
| Navegador HTTPS real con sesión | login público cargó; sesión no disponible | INCOMPLETO |

## Condiciones para una nueva evaluación

1. Comparar commits, imágenes, migraciones, grants y RLS en la VM DEV con el baseline local.
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
| 19. ¿Gaps bloqueantes? | Los tres HIGH fueron corregidos y revalidados localmente, no desplegados/medidos en VM; `AUD-007` recovery controller conserva impacto HIGH para piloto. |
| 20. ¿Riesgos aceptables? | Aún no aceptados formalmente: `AUD-001`, `AUD-002`, `AUD-003`, `AUD-009` y cobertura pendiente. |
