# Validación: cumplimiento de acuerdos

Fecha: 2026-11-18. Diseño: `AGREEMENT_FULFILMENT.md`.

Todo se ejecutó en la VM DEV, en el proyecto aislado `stir-af` (API en `127.0.0.1:38096`, web en `127.0.0.1:38189`). Se eliminó con `down -v --rmi local` y se comprobó que no quedaba nada: 0 contenedores, volúmenes, redes y directorios. stir-dev, ffm-dev y ffm-demo no se tocaron.

Ramas probadas: `claude/agreement-fulfilment-mvp` de stir-backend, stir-frontend, stir-main y stir-doc, partiendo de `main` tras la fusión de los términos estructurados.

## Resultados

| Prueba | Resultado |
|---|---|
| Backend `mvn verify` (PostgreSQL real, Flyway desde vacío hasta V23) | 306/306 |
| `FulfilmentRulesTest` | 3/3 |
| `FulfilmentPostgresTest` (como `idax_app`, bajo RLS) | 10/10 |
| `StirAuditCoveragePostgresTest`: las cuatro tablas nuevas clasificadas | 4/4 |
| Frontend `npm test`, `i18n:validate`, build | 60/60; 12 idiomas × 708 claves; build correcto |
| `agreement_fulfilment_e2e.py` (HTTP) | 33/33 |
| `agreement_fulfilment_browser.py` (Chromium, en castellano, sin errores de JavaScript) | PASS |
| Regresiones: `structured_terms_e2e`, `smoke`, `economic_exchange_e2e`, `negotiation_execution_integrity_e2e`, `marketplace_e2e`, `community_references_e2e`, `governed_boundary_e2e`, `community_value_governance_e2e`, `ordinary_governance_e2e` | todas PASS |

`audit_economic_commit_e2e.py` no se ejecutó: necesita el proxy de inyección de fallos (`STIR_AUDIT_FAULT_PROXY_URL`), que no forma parte del stack normal. Tampoco estaba en la batería de la 0.2.

## Lo que comprueba el E2E HTTP

- Un Agreement con términos estructurados produce cuatro obligaciones con sus partes. Se derivan una sola vez aunque lean dos personas.
- Una entrega parcial exige nota, queda `PARTIAL` y persiste con las dos declaraciones. La otra parte recibe `OBLIGATION_UPDATED`.
- Un "no recibido" de una sola parte queda `IN_DISAGREEMENT`. Pasa a `FAILED` sólo cuando el obligado dice `NOT_PERFORMED`.
- Nadie declara el lado de la otra parte, y nadie declara el intercambio económico. Una persona de la misma comunidad que no es parte recibe 404. Un superusuario de plataforma que no es parte tampoco puede declarar.
- Una enmienda: quien la propone no puede aceptarla, se encadena al snapshot, sustituye sólo lo que nombra y no reescribe el snapshot. Una base vieja y el intercambio económico se rechazan con 409.
- Los intentos de ejecución: un `commit` sin firmas queda `AUTHORISATION_INCOMPLETE`, después `COMMITTED`, y la obligación económica pasa a `DONE`. Un `commit` por encima del suelo queda `EXPOSURE_REFUSAL` con `CREDIT_FLOOR_EXCEEDED` en la base de datos, y la obligación económica sigue `PENDING`.
- `availableCapacity` es saldo menos suelo.
- La historia: no compartida por defecto. Una declaración sola no cuenta. Sólo cuenta lo confirmado por la otra parte, sin más campos que tipo, modo y número. Un recién llegado aparece como "sin historia todavía".

## Correcciones durante la validación

| Hallazgo | Corrección |
|---|---|
| Dos enmiendas creadas en la misma transacción compartían `now()` y su orden dependía del id. | Se insertan con `clock_timestamp()`. |
| El test de navegador usaba `check()` sobre una casilla controlada, que sólo cambia cuando STIR responde. | Usa `click()` y espera a que quede marcada. |
| En la captura, el título de cada tarjeta salía casi invisible: el estilo global de Shell pinta `header` de blanco. | La cabecera de la tarjeta es un `div` propio. Las notas se muestran tras un guion en lugar de entre comillas. |

## No verificado

- El despliegue en stir.es: todavía no tiene este incremento.
- `audit_economic_commit_e2e.py`, por la razón indicada arriba. El registro de intentos fallidos usa una transacción propia (`REQUIRES_NEW`), que pasa por el disparador de auditoría como cualquier otra escritura.
