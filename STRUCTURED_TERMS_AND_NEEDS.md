# Términos estructurados y necesidades

Estado: **rama `claude/structured-terms-and-needs-mvp`, sin merge** (2026-11-14).

Hasta ahora una publicación tenía título, descripción, categoría y lugar. Una propuesta tenía un mensaje, una cantidad, un importe y un texto libre de condiciones. Las personas que comparan y negocian necesitan distinguir más cosas:

- el estado de un objeto;
- cómo se entrega;
- qué incluye;
- hasta cuándo está disponible;
- qué puede cambiar en una necesidad y qué no;
- qué hace cada parte;
- qué queda abierto;
- hasta cuándo se puede aceptar una propuesta.

Este incremento hace que esas cosas se puedan expresar, conservar y aceptar de forma explícita. Todo es genérico, aditivo y opcional. Ninguna distribución aporta código, tablas ni semántica propios.

Nada de esto fija ni sugiere un precio. Son propiedades del objeto y del compromiso, no de su valoración.

## Publicaciones

`ListingRequest.details` (`ListingDetails`) es opcional y contiene:

| Campo | Valores | Uso |
|---|---|---|
| `itemCondition` | `NEW`, `LIKE_NEW`, `GOOD`, `FAIR`, `FOR_PARTS` | Estado de un objeto. Vacío si no aplica, por ejemplo en un servicio. |
| `fulfilmentMode` | `PICKUP`, `DELIVERY`, `PICKUP_OR_DELIVERY`, `ON_SITE`, `REMOTE` | Cómo cambia de manos o se presta. |
| `includedItems` | texto, máximo 1000 | Accesorios, materiales o límites de lo que se ofrece. |
| `availableUntil` | fecha | Fecha hasta la que está disponible. |
| `availability` | `AVAILABLE` (predeterminado), `IN_CONVERSATION`, `COMMITTED` | La distinción práctica entre "disponible para conversar", "en conversación" y "ya acordado". La declara quien publica y es independiente de `status` (`ACTIVE`/`CLOSED`). |
| `neededBy` | fecha, sólo en `WANTED` | Fecha antes de la cual se necesita. Lo garantizan la restricción `listing_needed_by_only_for_needs` y el servicio. |
| `flexibleOn`, `firmOn` | subconjuntos de `TIMING`, `PLACE`, `QUANTITY`, `CONDITION`, `AMOUNT` | Qué dimensiones de una necesidad pueden moverse y cuáles no. Una misma dimensión no puede ser flexible y firme a la vez. Se guardan en un orden canónico. |
| `counterpartNote` | texto, máximo 2000 | Detalle que quien publica comparte sólo con las personas que negocian. |

### La nota para la contraparte

Una necesidad puede requerir contexto, por ejemplo "tercer piso, sin ascensor". Ese contexto no tiene por qué ser público.

- `GET /listings/{id}` devuelve `counterpartNote` sólo al dueño y a quien tiene una negociación, en cualquier estado, sobre esa publicación.
- Los demás reciben `counterpartNote: null` y `hasCounterpartNote: true`. Así saben que hay más detalle sin poder leerlo.
- La búsqueda nunca devuelve la nota, tampoco al dueño.
- `ListingRevision` (formato `STIR-LISTING-REVISION-JCS-2`) congela los detalles comparables, pero no la nota. Las revisiones alimentan las observaciones `LISTING`/`WANTED` de referencias de valor, y la nota no debe llegar a la evidencia.

La nota no es un canal cifrado. La protegen la autorización del servicio y la RLS por tenant, como al resto de datos de STIR.

### Compatibilidad

Si un `PUT` no incluye `details`, se conservan los detalles existentes. Un cliente que no los conoce no los borra al editar el título. Si el `PUT` los incluye, los reemplaza.

Las revisiones anteriores conservan su formato `-1` y su digest.

## Propuestas

`OfferRequest.structuredTerms` (`OfferTerms`) también es opcional. Se acepta en la apertura y en las contraofertas.

| Campo | Uso |
|---|---|
| `scope` | Qué cubre y qué no. |
| `ownerTasks`, `initiatorTasks` | Qué hace quien publicó y qué hace quien abrió la negociación. Son los roles de la negociación, independientes de `OFFER`/`WANTED` y de quién paga. |
| `fulfilmentMode`, `fulfilBy` | Cómo y para cuándo se cumple. |
| `openQuestions` | Lo que las partes dejan abierto a sabiendas. |
| `validUntil` | Hasta cuándo se puede aceptar. Debe ser posterior al momento en que se hace la propuesta: una propuesta ya caducada se rechaza con 400. |

### Aceptar

- **Validez.** Si `validUntil` ya pasó, `accept` responde 409. La propuesta queda en la historia como `PROPOSED` y la otra parte puede contestar con una nueva.
- **Preguntas abiertas.** Si la propuesta tiene `openQuestions`, `accept` exige `acknowledgeOpenQuestions: true` (si no, responde 409). Aceptar significa aceptar estos términos con esas preguntas abiertas. Un acuerdo parcial es posible, pero tiene que ser consciente.

### Snapshot `schemaVersion = 4`

El snapshot añade `scope`, `ownerTasks`, `initiatorTasks`, `fulfilmentMode`, `fulfilBy`, `openQuestions`, `openQuestionsAcknowledged` y `offerValidUntil`.

- Los snapshots ya congelados con la versión 3 no cambian: ni bytes ni digest.
- Una propuesta sin términos estructurados produce un snapshot v4 con esos campos a `null` y `openQuestionsAcknowledged: false`.
- `GET /api/stir/instance` informa `externalContractSchemaVersion: 4`. Así una distribución sabe que puede usar estos campos.

`CATALOG_CONTRACT_VERSION` sigue en 1. Como en cambios anteriores, los añadidos son aditivos y las entradas previas siguen funcionando.

## Base de datos

`V22__structured_terms_and_needs.sql` sólo añade columnas: nueve a `stir.listing` (una con valor predeterminado) y siete a `stir.offer`, con restricciones `CHECK`. No crea tablas, no cambia permisos ni RLS, y el disparador de auditoría existente cubre las columnas nuevas.

Se numera V22 porque V20 y V21 pertenecen a la línea de vigencia de estado gobernado (`claude/governed-state-effectivity-boundary`). Una base que ya aplicó V22 sin V20 y V21 necesitará `outOfOrder` cuando esa línea se integre, o integrarla antes. Hay que decidirlo al hacer el merge.

## Lo que no hace

- No convierte ninguna de estas propiedades en un ajuste de precio ni en un criterio de elegibilidad de evidencia. Usar `itemCondition` o `fulfilmentMode` para estratificar referencias sería otra decisión comunitaria, con su propia gobernanza.
- No registra el cumplimiento: entregado, parcial o incumplido. Eso corresponde al incremento siguiente.
- No permite modificar un Agreement ya aceptado.
- `availability` no se deriva automáticamente de las negociaciones. La declara quien publica porque sólo esa persona sabe si sigue escuchando propuestas.
