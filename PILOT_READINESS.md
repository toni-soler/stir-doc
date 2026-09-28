# Pilot readiness — STIR/osTRIS/IDAX

**Estado: auditoría en curso; dictamen final pendiente.** La evidencia actual sobre los cinco `main` locales ya incluye dos findings HIGH abiertos de privilegios SQL comunitarios (`AUD-004`, `AUD-006`). El acceso a la VM DEV para inventario de DB/despliegue aún no está disponible. No se atribuye automáticamente a la VM el mismo estado de migraciones y grants hasta medirlo allí.

Con los criterios solicitados, el `main` local **no se puede declarar PILOT READY** mientras esos HIGH sigan abiertos. La clasificación final entre `PILOT READY WITH CONDITIONS` y `NOT PILOT READY` dependerá de la corrección/revalidación de los límites constitucionales y de las pruebas DEV reales; ningún resultado localhost puede sustituir backup/restore, reinicio, tenant A/B y navegador HTTPS.

| Control | Evidencia actual | Estado |
|---|---|---|
| Baseline cinco repos STIR PC | HEAD/branch/status/remote registrados en `FULL_SYSTEM_AUDIT.md` | VERIFICADO en PC |
| Baseline VM y pins reales | endpoint público 200; SSH histórico rechazado | INCOMPLETO |
| Clean PostgreSQL V1–V16 + backend | Testcontainers PostgreSQL 17.11; 216 tests PASS | VERIFICADO localmente |
| Autorización de rol DB comunitario | inserts `idax_admin` reproducidos en constitución e integridad | FAIL; HIGH abiertos |
| Tenant A/B y SuperAdmin HTTP/DB VM | fixtures y credenciales pendientes | NOT RUN |
| osTRIS económico, no-FX, extensión externa | inspección parcial de código/pin | INCOMPLETO |
| WebAuthn en VM, virtual y hardware manual | suite local existente; sesión VM pendiente | INCOMPLETO |
| Upgrade desde versión anterior con datos | pendiente | NOT RUN |
| Reinicios y durabilidad VM | pendiente | NOT RUN |
| Backup/restore a entorno aislado | procedimiento seguro en `AUDIT_RUNBOOK.md`; ejecución pendiente | NOT RUN |
| Navegador HTTPS real con sesión | login público cargó; sesión no disponible | INCOMPLETO |

## Condiciones para cerrar el dictamen

1. Comparar commits, imágenes, migraciones, grants y RLS en la VM DEV con el baseline local.
2. Registrar/corregir/revalidar cada HIGH técnico en rama específica y en upgrade real, sin alterar 7-of-7 ni introducir poderes Guardian.
3. Completar intentos adversariales `AUDIT-A`/`AUDIT-B`, WebAuthn, osTRIS, extensión, concurrencia e historial en VM.
4. Demostrar reinicios y restore real en DB/objetos separados; verificar histórico, referencias, Agreements, credenciales públicas y fotos.
5. Resolver o aceptar formalmente los `SPEC GAP`/`PRODUCT DECISION` según alcance de piloto. La fiscal valuation transaccional, si aparece, nunca será paridad osTRIS/fiat ni ingreso imponible/cuota fiscal del vendedor.

No hay score ni porcentaje. La tabla distingue pruebas realizadas de las pendientes.
