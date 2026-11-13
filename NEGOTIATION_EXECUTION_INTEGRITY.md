# Integridad de negociación y ejecución económica

Estado: **rama `claude/negotiation-execution-integrity-mvp`, sin merge** (2026-11-13). Corrige en STIR tres problemas genéricos que una re-auditoría independiente encontró desde un consumidor externo. Ningún problema depende de ese consumidor: los tres se reproducen sólo con las APIs de STIR (`stir-main/scripts/negotiation_execution_integrity_e2e.py`). No se añade código, tabla ni semántica de ninguna distribución a STIR u osTRIS.

## Clasificación y responsable

| # | Problema | Severidad | Responsable | Requiere |
|---|---|---|---|---|
| 1 | El pagador/receptor de un Agreement salía de la dirección **actual** del anuncio (`OFFER`/`WANTED`), que su dueño podía cambiar mientras el otro participante ya revisaba una oferta. El `offerId` y la `version` de la negociación no cambian, así que el contraparte aceptaba sin saber que se había invertido quién paga a quién. | Alta | STIR (`ListingService`, `NegotiationService`) | Sólo API pública |
| 2 | STIR no comprobaba que la transacción osTRIS apuntada por `stir.trade` fuese la de **ese** Agreement. Con la fila alterada, firmó/confirmó/informó como ejecución de A una transacción válida autorizada para B (mismas partes e importe). | Media | STIR (`TradeService`) | Escritura directa en la base de datos de STIR |
| 3 | Aperturas o contraofertas simultáneas terminaban en violaciones de restricción únicas devueltas como 5xx en vez de un conflicto determinista. | Baja | STIR (`NegotiationService`) | Sólo API pública |

El problema 2 **no** es una ruptura de las firmas de osTRIS: osTRIS verificó correctamente las firmas reales de B. Lo que faltaba era la correlación Agreement ↔ trade ↔ transacción en STIR. Exige poder escribir `stir.trade` (credencial de runtime o equivalente), el mismo límite de confianza que [`AUD-012`](FULL_SYSTEM_AUDIT.md); la corrección no cierra `AUD-012`, impide que esa alteración concreta se convierta en una ejecución atribuida a otro Agreement. No se ha encontrado defecto de osTRIS ni se añade semántica STIR a osTRIS.

## Reproducción sin el consumidor

Stack aislado en la VM DEV (`stir-main@cdc87e8` + `stir-backend@bc455fa` + `stir-frontend@9fb2211`, proyecto `stir-nei-base`, retirado con `down -v`). El script imprime `REPRODUCED`/`FIXED` por comprobación y sale con 1 si queda algo reproducido.

| Comprobación | Antes de la corrección |
|---|---|
| 1a. La negociación dice quién paga y quién recibe | REPRODUCED (no lo expone) |
| 1b. El dueño no puede cambiar OFFER/WANTED con una negociación abierta | REPRODUCED (HTTP 200) |
| 1c. Aceptar nunca invierte pagador/receptor | REPRODUCED: el aceptante pasó a pagador con `offerId`/`version` idénticos |
| 2a–2d (transacción y digest sustituidos) | REPRODUCED: `signing-payload` 200, `commit` informó COMMITTED, `sync` 200, Agreement A en COMMITTED |
| 2a–2d (sólo transacción sustituida) | REPRODUCED, mismos resultados |
| 3a. Aperturas simultáneas | No reproducido en esta ejecución (`[200, 409×5]`); depende del orden: la comprobación previa suele ganar la carrera |
| 3b. Contraofertas simultáneas | REPRODUCED: `[200, 500×5]`, `duplicate key ... offer_negotiation_id_sequence_number_key`; sin ofertas parciales |
| 3c. Aceptaciones simultáneas | No reproducido (`[200, 409×5]`, un Agreement) |

Total: 12 de 14 comprobaciones reproducidas.

## Corrección

### 1. La dirección económica forma parte de lo aceptado

- `ListingService.update` toma un bloqueo de fila (`lockForUpdate`) y rechaza con 409 cambiar `direction` mientras exista una negociación `OPEN` sobre el anuncio. Los demás campos se siguen editando igual.
- `NegotiationService.open` y `accept` leen el anuncio con bloqueo compartido (`lockForShare`), de modo que una edición del dueño no puede intercalarse ni con la apertura ni con la aceptación.
- `accept` deriva pagador/receptor de la dirección vigente cuando se abrió la negociación. Se usan las revisiones inmutables que ya existían (`ListingRevision`, [Multi-Source Value Evidence](MULTI_SOURCE_VALUE_EVIDENCE.md)): si la revisión vigente en `negotiation.createdAt` no coincide con la dirección actual (datos de antes de esta corrección), se rechaza con 409 en vez de reinterpretar. Sin revisión registrada se conserva el comportamiento anterior.
- `NegotiationDetail` expone `listingDirection`, `payerUserId` y `payeeUserId`; `AgreementDetail` expone `payerUserId`/`payeeUserId` (ya estaban en el snapshot).
- `POST .../negotiations/{id}/accept` acepta `expectedPayerUserId`/`expectedPayeeUserId` opcionales: si se envían y no coinciden con los roles que muestra `NegotiationDetail`, 409. Los roles se comprueban también en ofertas sin importe; el Agreement sólo congela `payerUserId`/`payeeUserId` cuando la oferta aceptada lleva importe. La UI de STIR muestra "si aceptas, pagas/recibes este importe" (12 idiomas) y envía esas dos partes.
- No se reinterpreta ningún Agreement ya aceptado: su snapshot sigue siendo la fuente.

### 2. Correlación Agreement ↔ trade ↔ intención osTRIS

`TradeService.requireIntent` compara, antes de `signing-payload`, `authorize`, `commit` (incluido el caso ya confirmado) y `sync`, la transacción que osTRIS devuelve con la intención re-derivada de datos propios de STIR, nunca de la fila `stir.trade`:

- `transactionId` (fila, estado y payload), comunidad y unidad del binding del marketplace, `purpose=EXCHANGE`;
- `contractualMetadataDigest` = digest del snapshot del Agreement;
- exactamente dos entradas: cuenta del pagador `-importe`, cuenta del receptor `+importe`, con cuentas de los bindings de los participantes e importe de la oferta aceptada en unidades menores;
- la propia fila `stir.trade` (partes, cuentas, importe, digest) coherente con lo anterior.

Si algo difiere: 409 `EXECUTION_INTENT_MISMATCH`, no se relaya firma, no se llama a `commit`, no cambia ningún estado y se registra un aviso con los campos discrepantes. La idempotencia (commit repetido, reconciliación por `sync`, reparación del falso `REJECTED` heredado) se conserva: el commit usa una sola lectura de estado para verificar y reconciliar.

### 3. Conflictos de concurrencia normalizados

`Conflicts.onLostRace` traduce a 409 sólo las restricciones que significan "otra petición llegó antes" (`negotiation_one_open_thread`, `offer_negotiation_id_sequence_number_key`, `agreement_negotiation_id_key`) y los fallos de bloqueo optimista de la negociación. Cualquier otra violación sigue propagándose como error. Todo ocurre dentro de la transacción, que se revierte: no quedan ofertas parciales ni Agreements duplicados.

## Compatibilidad

- API aditiva: campos nuevos en respuestas y dos campos opcionales en `accept`. Los clientes que no los envían funcionan igual.
- Cambio de comportamiento deliberado: cambiar la dirección de un anuncio con negociaciones abiertas devuelve 409.
- `signing-payload`, `authorize`, `commit` y `sync` hacen una lectura adicional a osTRIS (`GET /transactions/{id}`, y la unidad para la escala).
- Sin migraciones: no colisiona con las `V20`/`V21` de la rama `claude/governed-state-effectivity-boundary`.

## Límites y pendientes

- Detectar la alteración de `stir.trade` no impide que quien tenga escritura en la base la haga; sigue abierto `AUD-012`.
- Negociaciones abiertas sobre anuncios cuyo cambio de dirección anterior no dejó revisión (anteriores a `V15`) no pueden detectarse; se aceptan con la dirección actual como antes.
- No se ha comprobado qué revisión está desplegada en stir.es; no se afirma que esté afectada ni corregida.

## Validación

Ver [VALIDATION_NEGOTIATION_EXECUTION_INTEGRITY.md](VALIDATION_NEGOTIATION_EXECUTION_INTEGRITY.md).
