# Validación — términos estructurados y necesidades

Fecha: 2026-11-14. Todo se ejecutó en la VM DEV, en proyectos compose aislados (`stir-m02`, `ffm-m02`) que se retiraron con `down -v --rmi local` (`left=0`). No se tocó stir-dev ni ningún otro stack compartido.

## Revisiones

| Repo | Rama `claude/structured-terms-and-needs-mvp` |
|---|---|
| stir-backend | `1074984` |
| stir-frontend | `a98a89c` |
| stir-main | `9dbdc9a` |
| stir-doc | este documento |

La validación de extremo a extremo no se hizo con estas ramas solas. `stir-main` `main` provisiona el verificador de estado gobernado, que necesita las migraciones V20 y V21 de la línea de vigencia. Por eso el stack se construyó con una combinación local sin publicar:

- línea de vigencia;
- integridad de negociación (ya en `main`);
- esta rama.

Las revisiones combinadas son backend `8b5de98` y frontend `39efc55`, sin conflictos de merge. Es lo que correría stir-dev. Con el backend de `main` a secas, el stack no arranca: `governed-verifier-provision` falla con `role "idax_governed_verifier" does not exist`. Esa limitación ya existía; este incremento no la introduce.

## Resultados

| Prueba | Resultado |
|---|---|
| `mvn verify`, rama sola | 255/255 (243 previos + 12 nuevos); Flyway desde vacío hasta V22 |
| `mvn verify`, combinación | 290/290; V20, V21 y V22 en orden |
| Frontend (Node 20) | `npm test` 50/50; `i18n:validate` 12 × 610; build correcto |
| `structured_terms_e2e.py` | 19/19 |
| `negotiation_execution_integrity_e2e.py` | 16/16 FIXED |
| `marketplace_e2e.py`, `economic_exchange_e2e.py`, `community_references_e2e.py` | PASS |
| FreeFolk Market sobre esta STIR (consumidor, repo privado) | E2E HTTP 74/74; navegador 32/32 |

## Qué cubre `structured_terms_e2e.py`

- **1a–1d.** Una necesidad conserva su fecha y sus dimensiones flexibles y firmes. Se rechazan:
  - una fecha de necesidad en una oferta;
  - una dimensión que sea flexible y firme a la vez;
  - un estado desconocido.
- **2a–2f.** La nota para la contraparte:
  - la lee el dueño;
  - un tercero solo sabe que existe;
  - nunca aparece en la búsqueda;
  - quien negocia la lee y quien no negocia, no;
  - la revisión congela los detalles comparables (JCS-2), pero no la nota.
- **3.** Una edición sin `details` los conserva.
- **4a–4c.** La instancia declara el esquema 4. El snapshot congela alcance, tareas de cada lado y plazo. Una oferta sin términos estructurados se acepta como antes.
- **5a–5e.** Las preguntas abiertas:
  - sin reconocimiento expreso, la aceptación da 409;
  - con reconocimiento, el snapshot lo registra.

  La validez de la oferta:
  - una oferta caducada se rechaza con 409, y la conversación sigue con una nueva;
  - no se puede crear una oferta ya caducada (400).

## Pendiente de decidir al hacer merge

La numeración V22 supone que la línea de vigencia (V20/V21) llegará a `main`. Si V22 entra antes en una base desplegada, Flyway necesitará `outOfOrder` para aplicar V20/V21 después. Ver `STRUCTURED_TERMS_AND_NEEDS.md`.
