# Pilot readiness — STIR/osTRIS/IDAX

**Dictamen del baseline `main` auditado: NOT PILOT READY.** Hay tres findings HIGH abiertos en `main`: el rol SQL administrativo puede insertar constitución no firmada (`AUD-004`), fabricar un FINAL de Market Integrity (`AUD-006`) y STIR vuelve terminal un commit osTRIS pendiente ante 503 o firmas incompletas (`AUD-010`). Los dos primeros vulneran el límite de autoridad comunitaria/constitucional en la base y el tercero la integridad del estado económico. Según los criterios de esta auditoría, cualquiera impide PILOT READY mientras siga abierto.

**Alcance del dictamen:** código y migraciones de los cinco `main` locales, pins vendorizados y sondas públicas limitadas a `https://dev.stir.es`. El acceso a la VM DEV para inventario de DB/despliegue aún no está disponible. No se atribuye automáticamente a la VM el mismo estado de migraciones y grants hasta medirlo allí. La auditoría de VM, tenant A/B, backup/restore y navegador autenticado sigue inconclusa; podría descubrir más blockers, pero no convertir el `main` ya fallido en apto. Las ramas de fix no alteran este dictamen hasta integrarse y revalidarse.

| Control | Evidencia actual | Estado |
|---|---|---|
| Baseline cinco repos STIR PC | HEAD/branch/status/remote registrados en `FULL_SYSTEM_AUDIT.md` | VERIFICADO en PC |
| Baseline VM y pins reales | endpoint público 200; SSH histórico rechazado | INCOMPLETO |
| Clean PostgreSQL V1–V16 + backend | Testcontainers PostgreSQL 17.11; 216 tests PASS | VERIFICADO localmente |
| Autorización de rol DB comunitario | inserts `idax_admin` reproducidos en constitución e integridad | FAIL; HIGH abiertos |
| Commit económico ante 503 osTRIS | 503 provoca `REJECTED` definitivo (`AUD-010`) | FAIL; HIGH abierto |
| Tenant A/B y SuperAdmin HTTP/DB VM | fixtures y credenciales pendientes | NOT RUN |
| osTRIS económico, no-FX, extensión externa | inspección parcial de código/pin | INCOMPLETO |
| WebAuthn en VM, virtual y hardware manual | suite local existente; sesión VM pendiente | INCOMPLETO |
| WebAuthn malformado | `AUD-008` reproducido pre-fix; fix `8e4003a` validado localmente | FIX LOCAL; VM/main pendientes |
| Object storage | digest reproducible pero anterior a releases con fixes (`AUD-009`) | REVISIÓN/RESTORE pendientes |
| Upgrade desde versión anterior con datos | pendiente | NOT RUN |
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
| 2. ¿Puede Platform SuperAdmin gobernar? | **FAIL en límite DB:** `idax_admin` inserta constitución/finding. No se ha probado explotación HTTP por SuperAdmin. |
| 3. ¿Guardian octava llave? | Tests locales rechazan sustitución de firma; ceremonia VM y concurrencia pendientes. |
| 4. ¿6-of-7 constitucional? | Tests locales rechazan amendment 6-of-7; VM pendiente. La excepción documentada 5-of-7 sólo remueve Guardian. |
| 5–6. ¿Replay/cross-community WebAuthn? | Tests locales cubren ambos rechazos; browser VM pendiente. COSE malformada reveló `AUD-008`. |
| 7. ¿Withdrawal borra evidencia? | Tests locales preservan snapshot; `AUD-002` revela hold potencialmente indefinido tras DISMISSED. VM pendiente. |
| 8–10. ¿Cuentas, lineage y seed inflan evidencia? | Cobertura unitaria existente, pero ataque activo con cuentas/relists/seed en VM pendiente. |
| 11. ¿Extensión salta invariantes? | SQL `idax_admin` sí salta dos; contrato público externo y permisos VM aún sin prueba activa. |
| 12. ¿Se modifica external contract aceptado? | Snapshot/digest en código; replay y mutación adversarial VM pendientes. |
| 13. ¿STIR fuerza osTRIS? | Cliente actual sólo `EXCHANGE` con firmas de cuentas; `AUD-010` muestra manejo incorrecto de fallos, no force commit. E2E pendiente. |
| 14. ¿Paridad fiat/osTRIS? | Búsqueda estática acotada sin paridad; runtime/contratos externos pendientes. |
| 15. ¿Migraciones de DB limpia? | PASS local Testcontainers PostgreSQL 17.11 para STIR V1–V16; stack completo/VM pendiente. |
| 16–17. ¿Reinicio/restore? | NOT RUN en VM. Restore debe usar DB y volúmenes separados. |
| 18. ¿HTTPS/browser VM? | Login público y health GET cargan; sesión y recorridos E2E NOT RUN. |
| 19. ¿Gaps bloqueantes? | `AUD-004`, `AUD-006`, `AUD-010` HIGH; `AUD-007` recovery controller de alto impacto. |
| 20. ¿Riesgos aceptables? | Aún no aceptados formalmente: `AUD-001`, `AUD-002`, `AUD-003`, `AUD-009` y cobertura pendiente. |
