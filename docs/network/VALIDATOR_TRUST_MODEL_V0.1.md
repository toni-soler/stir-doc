# Validator trust model v0.1

**Estado:** propuesta, sin alta de validators ni claves. La UNL de XRPL es una decisión local de cada servidor; un publisher recomienda una lista firmada y no adquiere autoridad constitucional sobre STIR. Las listas usadas por nodos de la misma comunidad necesitan solapamiento alto para evitar divergencias. [Estructura del consenso](https://xrpl.org/docs/concepts/consensus-protocol/consensus-structure), [protecciones y solapamiento](https://xrpl.org/docs/concepts/consensus-protocol/consensus-protections).

## Identidades y custodia

`operator_id` (persona/organización y contacto verificado) ≠ clave maestra pública del validator ≠ clave efímera activa de firma ≠ hostname/IP de servidor ≠ clave del publisher. El registro público vincula operador con clave maestra por una declaración firmada y revisable; cambiar host no cambia esa clave. Verificar control de contacto/dominio ayuda a acreditar operador, pero DNS no constituye identidad permanente ni prueba independencia humana.

En una futura ceremonia controlada se genera cada `validator-keys.json` fuera del host validator con `validator-keys create_keys`, se cifra y respalda offline; el token activo se obtiene con `create_token` y sólo éste llega al host, con permisos restrictivos. El token **también es secreto**. Tras crear un token, actualizar el backup maestro para conservar la secuencia: generar varios desde una copia vieja puede repetir `token_sequence`. La clave maestra firma manifiestos de rotación; la clave efímera firma validaciones. Una rotación de token conserva la identidad maestra; un host migrado usa token nuevo y el anterior se deshabilita. Si se compromete el host, aislarlo, comparar hashes/validaciones, rotar token desde la maestra offline y reconstruir en un host limpio. Si se compromete la maestra, revocarla definitivamente, excluir la identidad de la UNL y admitir una identidad nueva tras revisión; no afirmar continuidad criptográfica de la identidad revocada. [Guía de validator](https://xrpl.org/docs/infrastructure/configuration/server-modes/run-xrpld-as-a-validator), [manifests/lista](https://xrpl.org/docs/references/http-websocket-apis/peer-port-methods/validator-list). HSM es opción futura, no requisito de la demo.

## Lista y publisher

**Phase A:** un publisher del proyecto, con clave propia offline para emisión controlada, distribución en dos ubicaciones, secuencia monotónica, expiración finita y cambios registrados con razón, aprobación y digest anterior/nuevo. El proceso firma un blob/lista en el formato soportado por `xrpld 3.3.0`; una copia web no puede alterar la firma. Nodos fijan la clave pública del publisher por un canal de confianza fuera de DNS y monitorizan `validators`, `validator_list_sites`, secuencia, expiración, conjunto resultante y quorum. Una réplica de contenido firmado resuelve caída de hosting; si se pierde la clave del publisher hay que efectuar una transición explícita de confianza en la configuración de los nodos. No suponer renovación automática de lista expirada. [Estado de listas](https://xrpl.org/docs/references/http-websocket-apis/admin-api-methods/status-and-debugging-methods/validators), [sitios](https://xrpl.org/docs/references/http-websocket-apis/admin-api-methods/status-and-debugging-methods/validator_list_sites).

**Phase B:** comité de confianza de operadores y usuarios técnicos independientes aprueba altas/bajas; dos publishers de distintas custodias pueden emitir listas coherentes, con política documentada de intersección, expiración y transición. La configuración concreta de umbral debe probarse en 3.3.0: el valor por defecto para 1–2 publisher keys es 1, y para más es `floor(N/2)+1`; esto **no** es automáticamente un multisig de gobernanza. Publicar listas dispares o cambiar claves sin coordinación puede dejar conjuntos distintos entre nodos. [Umbral de listas](https://xrpl.org/docs/infrastructure/configuration/configure-validator-list-threshold).

**Phase C:** organizaciones independientes custodian publishers y operadores eligen explícitamente las claves de lista que siguen. Cambios de conjunto requieren aviso, ventana de solapamiento verificada y ensayo de pérdida de publisher. La autoridad para publicar una lista es distinta de la decisión colectiva sobre su contenido; múltiples sitios con la misma clave sólo dan disponibilidad, no diversidad de gobernanza.

Una baja ordinaria se publica con motivo, secuencia mayor y expiración adecuada. Para compromiso grave: aviso por canal fuera de banda, lista firmada de emergencia sin la clave afectada, verificación desde al menos dos nodos, y revocación de la master key si procede. Evitar listas estáticas de emergencia indefinidas: sólo como procedimiento auditado y ensayado. Si el publisher no está disponible, continuar mientras la lista válida y replicada lo permita; antes de expirar, recuperar la custodia o coordinar nueva clave/lista en cada nodo. No cambiar el conjunto confiado a ciegas para restablecer liveness.

## Cuatro gobiernos separados

| Dominio | Decide | No decide automáticamente |
|---|---|---|
| Económico/comunitario | normas económicas, membresía, mercado osTRIS/STIR | UNL o acceso al host |
| Constitucional/integridad | cambios protegidos de STIR y sus Seven Keys | aceptación de validators XRPL |
| Validator trust | criterios y lista recomendada, altas/bajas | política económica, administración de servidores |
| Administración de infraestructura | VMs, red, DNS, backups, parches | quién merece voto confiado |

Las Seven Keys existentes no firman ni gobiernan validators por defecto: fueron definidas para autoridad constitucional de STIR. Reutilizarlas requeriría decisión constitucional explícita, análisis de liveness/captura y nuevo contrato; no se deduce de compartir comunidad. [Gobernanza constitucional de STIR](../../SEVEN_KEYS_GOVERNANCE.md).
