# Cumplimiento de acuerdos: obligaciones, confirmaciones, enmiendas y resultados

Estado: **implementado** en las ramas `claude/agreement-fulfilment-mvp` de stir-backend, stir-frontend, stir-main y stir-doc (2026-11-18). La validación está en `VALIDATION_AGREEMENT_FULFILMENT.md`.

Un Agreement de STIR dice qué se acordó, pero no si se cumplió. Hoy su único estado posterior es la fase económica:

- `AWAITING_ECONOMIC_EXECUTION`;
- `AWAITING_SIGNATURES`;
- `COMMITTED`;
- `NOT_APPLICABLE`.

Un intercambio puede tener el tramo económico confirmado y el objeto sin entregar, o al revés. Un intento de ejecución rechazado por el suelo de crédito sólo existe como respuesta HTTP: no queda registrado.

Este incremento separa tres cosas que hoy no se distinguen:

- **acuerdo**: el snapshot congelado, que no cambia;
- **cumplimiento**: qué obligación se hizo, quién lo dice y qué confirma;
- **registro**: el asiento de osTRIS, que sigue siendo la única verdad económica.

Todo es genérico. Ninguna distribución aporta código, tablas ni semántica. FIAT, comisiones y proveedores de pago no existen aquí. Una distribución con su propio tramo externo, como el EUR de FreeFolk Market, lo sigue en su propio sistema y lo muestra junto a estas obligaciones.

## Obligaciones

Al crear un Agreement, STIR deriva sus obligaciones del snapshot de forma determinista. Para los Agreements anteriores a este incremento, la derivación se hace la primera vez que se leen. Ningún snapshot cambia: las obligaciones se guardan en una tabla aparte que apunta al snapshot.

| `kind` | Origen en el snapshot | Obligado | Beneficiario |
|---|---|---|---|
| `OBJECT` | La publicación y `scope` | Quien aporta el bien o servicio: el dueño en `OFFER`, quien inicia en `WANTED` | La otra parte |
| `OWNER_TASKS` | `ownerTasks`, si no está vacío | El dueño de la publicación | Quien inicia |
| `INITIATOR_TASKS` | `initiatorTasks`, si no está vacío | Quien inicia | El dueño |
| `ECONOMIC_EXCHANGE` | `proposedAmount`, si existe | El pagador (`payerUserId`) | El cobrador (`payeeUserId`) |

Cada obligación guarda `fulfilBy` y `fulfilmentMode` del snapshot, o de la última enmienda aceptada que la sustituyó.

### Estados

| Estado | Significado |
|---|---|
| `PENDING` | Nadie ha declarado nada todavía. |
| `DECLARED` | El obligado ha declarado algo (que la cumplió, en parte o que no). Falta lo que diga el beneficiario. |
| `PARTIAL` | El beneficiario confirma una parte. Lo que falta se describe en texto. |
| `DONE` | El beneficiario confirma la conformidad. |
| `IN_DISAGREEMENT` | Las declaraciones de las partes se contradicen. |
| `FAILED` | Las dos partes coinciden en que no se cumplió. |
| `SUPERSEDED` | Una enmienda aceptada la sustituyó. |

`FAILED` exige que las dos partes coincidan. Si una sola dice "no recibido", eso es una declaración, no una conclusión. El estado pasa a `IN_DISAGREEMENT`, y ese desacuerdo se resolverá en 0.4 con el caso de revisión. STIR nunca decide quién tiene razón y no sanciona.

La obligación `ECONOMIC_EXCHANGE` no se confirma a mano. Su estado se deriva del `Trade`:

- `COMMITTED` equivale a `DONE`;
- cualquier otro estado equivale a `PENDING`, acompañado del último resultado de ejecución (ver más abajo).

Una firma de osTRIS es la autorización económica, y osTRIS ya la registra. STIR no añade una segunda confirmación que pudiera contradecirla.

### Confirmaciones

Cada declaración es un evento que no se puede editar ni borrar (`stir.agreement_obligation_event`). Indica qué confirma:

| `type` | Quién | Qué confirma |
|---|---|---|
| `PERFORMED` | Obligado | Lo entregó o prestó. |
| `PARTIALLY_PERFORMED` | Obligado | Hizo una parte. El campo `note` es obligatorio. |
| `NOT_PERFORMED` | Obligado | No lo hizo. |
| `CONFORMITY_CONFIRMED` | Beneficiario | Lo recibió conforme a lo descrito. |
| `PARTIAL_RECEIVED` | Beneficiario | Recibió una parte. El campo `note` es obligatorio. |
| `NONCONFORMITY_REPORTED` | Beneficiario | Lo recibió, pero no como se describió. |
| `NOT_RECEIVED` | Beneficiario | No lo recibió. |

El estado es una función pura del último evento de cada parte. Una parte puede rectificar con un evento nuevo, y el anterior queda en la historia. Sólo las dos partes del Agreement pueden registrar eventos. Esto se comprueba en el servicio, como hace `TradeService.partyAgreement`. Un superusuario que no es parte recibe 404, aunque `@PreAuthorize` le dejaría pasar.

## Enmiendas

Una enmienda es un acto aceptado. Indica qué sustituye y qué conserva, y nunca reescribe el snapshot.

- Una parte propone. La propuesta contiene:
  - `baseDigest`: el digest del snapshot, o de la última enmienda aceptada;
  - `replaces`: los ids de las obligaciones que se sustituyen;
  - `obligations`: las obligaciones nuevas, con sus tareas, `fulfilBy` y `fulfilmentMode`;
  - `reason`;
  - `validUntil`.
- La otra parte acepta o rechaza. Si `baseDigest` ya no es el último, aceptar responde 409: alguien enmendó antes.
- Al aceptar se congela un `AgreementAmendment` con formato `STIR-AGREEMENT-AMENDMENT-JCS-1`. Su digest encadena el anterior. Las obligaciones sustituidas pasan a `SUPERSEDED`, y las demás se conservan sin cambios.
- No se puede sustituir una obligación en estado `DONE` ni `FAILED`, ni la obligación `ECONOMIC_EXCHANGE`. Cambiar importe, unidad, partes o compromiso externo (`externalContractDigest`) queda fuera. En osTRIS eso es una corrección con base final, y corresponde a 0.4.

## Resultados de ejecución

Cada intento de `commit` hacia osTRIS queda registrado en `stir.economic_execution_attempt`, también cuando falla. Se graba en una transacción propia, para que la excepción que se devuelve al cliente no lo deshaga. El resultado se clasifica así:

| `outcome` | Códigos de osTRIS o situación | Lectura |
|---|---|---|
| `COMMITTED` | Recibo de osTRIS | Ejecutado. |
| `EXPOSURE_REFUSAL` | `CREDIT_FLOOR_EXCEEDED` | La restricción funcionó como estaba diseñada. No es un juicio sobre el precio. |
| `AUTHORISATION_INCOMPLETE` | `CONTROL_POLICY_NOT_SATISFIED` | Faltan firmas. También es una restricción que funcionó. |
| `INVALID_DESCRIPTION` | `ACCOUNT_SCOPE_MISMATCH`, `INVALID_AMOUNT`, `AMOUNT_TOO_LARGE`, `NON_ZERO_SUM`, `PROPOSAL_INTENT_MISMATCH` | Lo que se pidió ejecutar está mal descrito. |
| `IMPLEMENTATION_FAULT` | 400/404 de osTRIS sobre una transacción local que existe; `PROTOCOL_DIGEST_MISMATCH`; `PROJECTION_DRIFT` | Fallo del sistema, no de las personas. |
| `UNKNOWN` | Timeout, 429, 5xx o red; cualquier código no clasificado | No se sabe. `sync()` lo resolverá. |

Un código que no está en la tabla se clasifica como `UNKNOWN`, nunca como uno de los otros resultados. Así una clasificación nunca afirma más de lo que se sabe. El intento guarda también el código original de osTRIS.

Un rechazo no deja la obligación en `FAILED`. Política y crédito pueden cambiar, y el Trade sigue siendo reintentable, como hoy.

`TRANSACTION_LIFECYCLE.md` todavía dice que un fallo deja el Trade `REJECTED`. El código actual ya no lo hace: lo mantiene reintentable y sólo repara el estado heredado. Este incremento corrige ese documento.

## Crédito

`GET /economic/me` ya devuelve `balanceProjection` y `creditFloor` desde osTRIS. Se añade `availableCapacity` (saldo − suelo, mínimo 0), calculada igual que `CreditPosition.availableNegativeExposure()` de osTRIS. Así ninguna distribución tiene que hacer la resta.

**Hueco que queda abierto (R10):** en osTRIS el suelo es un número fijado al activar la cuenta. No tiene autor ni procedimiento de revisión. Este incremento lo muestra tal cual es, "fijado al activar", sin inventar un autor. Gobernar el suelo es una decisión de política comunitaria que pertenece a 0.5.

## Historia contextual

`GET /participants/{userId}/fulfilment-history` devuelve hechos descritos y ninguna puntuación. Por ejemplo, "completó 3 entregas descritas, 1 servicio presencial". Las reglas:

- Sólo cuenta obligaciones `DONE` confirmadas por la otra parte. No incluye importes, contrapartes, fechas exactas ni ratios.
- **Es opcional y está desactivada por defecto** (`ParticipantProfile.shareFulfilmentHistory`). En comunidades pequeñas, una historia visible permite inferir con quién trata cada persona (R17).
- La respuesta distingue `NO_HISTORY_YET` de `NOT_SHARED`. Un recién llegado nunca aparece "bajo".
- No se calcula ni expone ninguna puntuación, porcentaje de éxito ni contador de fallos.

## API

Todas las rutas van bajo `/api/stir/tenants/{tenantId}/agreements/{id}`:

- `GET /obligations`: devuelve las obligaciones con su estado, sus eventos y el último resultado económico.
- `POST /obligations/{obligationId}/events`.
- `POST /amendments`.
- `POST /amendments/{amendmentId}/accept`.
- `POST /amendments/{amendmentId}/reject`.
- `GET /amendments`.
- `GET /trade/attempts`.

Y bajo `/api/stir/tenants/{tenantId}/participants`:

- `GET /{userId}/fulfilment-history`.
- `PUT /me/fulfilment-history-sharing` con `{ "share": true | false }`. Es una ruta propia y no un campo de `PUT /me`, para que un cliente que no la conoce no la desactive al editar su perfil. `GET /me` devuelve `shareFulfilmentHistory`.

La otra parte recibe una notificación con cada declaración (`OBLIGATION_UPDATED`) y con cada enmienda propuesta, aceptada o rechazada (`AMENDMENT_PROPOSED`, `AMENDMENT_ACCEPTED`, `AMENDMENT_REJECTED`).

Lectura con `stir.agreements.read` y escritura con un permiso nuevo, `stir.agreements.fulfil`, que se añade a `config/permission-catalog.json`. Los roles de participante los crea el administrador en Shell con una lista de permisos. Por eso el permiso tiene que añadirse a esa lista en los fixtures E2E, en el smoke y en la guía de alta de comunidades, igual que `stir.economic.manage`. `GET /api/stir/instance` añade `agreementFulfilmentVersion: 1`, para que una distribución sepa si puede usarlo.

## Base de datos

`V23__agreement_fulfilment.sql` crea:

- `stir.agreement_obligation`. Un disparador congela su identidad y sus términos: sólo pueden cambiar el estado derivado y la enmienda que la sustituyó, y una obligación sustituida no vuelve atrás. No se borra nunca.
- `stir.agreement_obligation_event`: sólo se puede insertar.
- `stir.agreement_amendment`. Una propuesta se decide una sola vez; después ninguna fila cambia. Dos índices únicos parciales hacen que sólo una enmienda aceptada pueda construirse sobre cada base y que su secuencia no se repita: si dos aceptaciones compiten, la base de datos decide y la perdedora recibe 409.
- `stir.economic_execution_attempt`: sólo se puede insertar.

Además añade `participant_profile.share_fulfilment_history` y los cuatro tipos de notificación nuevos.

Todas las tablas tienen RLS por tenant y se registran en `stir_audit.coverage_registry` como `COVERED` en el dominio `AGREEMENT_ECONOMIC`, con el disparador de auditoría y la lectura del auditor como las demás. `idax_app` sólo recibe `INSERT` en las tablas de historia; `idax_admin` sólo lectura. El disparador de sólo inserción actúa también contra el propietario de la tabla, no depende de los permisos.

Las obligaciones se derivan la primera vez que alguien las lee, también para los Agreements anteriores a V23. `NegotiationService.accept` no cambia. Una restricción única parcial por `(agreement_id, kind)` hace que dos lectores simultáneos no dupliquen nada.

## Pruebas

- **Backend:** `FulfilmentRulesTest` (función de estado y clasificación de cada código) y `FulfilmentPostgresTest`, contra PostgreSQL real como `idax_app` bajo RLS. Cubren:
  - la derivación de obligaciones y sus partes, en `OFFER` y en `WANTED`;
  - la regla de doble coincidencia de `FAILED`;
  - el encadenamiento de enmiendas, el 409 sobre una base vieja y las obligaciones que no se pueden sustituir;
  - la RLS entre tenants y el 404 a un superusuario que no es parte;
  - que ningún evento, intento, enmienda decidida u obligación sustituida se pueda modificar.
- **E2E HTTP** (`agreement_fulfilment_e2e.py`, en un stack aislado):
  - entrega parcial;
  - desacuerdo;
  - fallo con las dos partes de acuerdo;
  - enmienda que cambia una tarea y la fecha;
  - un intento sin firmas (`AUTHORISATION_INCOMPLETE`) seguido de `COMMITTED`;
  - un intento por encima del suelo, persistido como `EXPOSURE_REFUSAL`, con la obligación económica todavía `PENDING`;
  - la capacidad disponible;
  - historia `NOT_SHARED` frente a `NO_HISTORY_YET`.
- **Navegador** (`agreement_fulfilment_browser.py`): el tablero en el detalle del Agreement, una declaración parcial y una enmienda hechas desde la UI, la capacidad en la página económica y la historia en el perfil, en castellano y sin errores de JavaScript.
- **UI de STIR:** 12 idiomas, validados con `npm run i18n:validate`.

## Lo que no hace

- No resuelve desacuerdos, no repara efectos y no sanciona. Eso es 0.4.
- No enmienda el tramo económico ni el compromiso externo.
- No gobierna el suelo de crédito.
- No calcula reputación.
