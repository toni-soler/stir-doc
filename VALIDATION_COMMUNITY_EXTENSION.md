# Validation: primer consumidor comunitario del catálogo

Fecha: 2026-09-26. Alcance: `stir-frontend`, con composición temporal local bajo `stir-main/.local/community-e2e/` (ignorada por Git). No se cambió el backend ni ningún contrato de osTRIS/Ledger.

## Cambio comprobado

STIR y una segunda extensión de Shell consumen el mismo `src/catalog-client.js`. La segunda se compila importando `dist/community/stir-catalog.mjs`, usa su propio CSS y ruta `/community-catalog`, y entrega la negociación a la ruta existente de STIR. La composición habitual de STIR no incluye esa extensión de ejemplo.

## Evidencia ejecutada

| Comprobación | Resultado |
|---|---|
| `npm test` en `stir-frontend` | PASS: 38 pruebas, incluidas ruta tenant, cancelación, oferta, payload y error de conflicto del contrato de catálogo. |
| `npm run i18n:validate` | PASS: 12 locales y 438 claves por locale. No se añadieron claves. |
| `npm run build` | PASS: bundle STIR, ESM de catálogo y bundle de segunda presentación. El build importa el ESM generado, evalúa el registro de Shell y comprueba los 12 bundles de traducción. |
| `docker compose -f compose.yml -f .local/community-e2e/compose.yml up -d --build` | PASS: migraciones y servicios locales saludables con manifest/proxy temporales. |
| Playwright/Edge contra `http://localhost:8089` | PASS: login real, extensión comunitaria cargada, petición autenticada al catálogo real de STIR. Después, fixture interceptado **sólo en navegador** para comprobar detalle, POST de oferta y navegación a `/stir/negotiations/{id}` sin escribir un anuncio o negociación en la base de datos. |
| `docker compose ... down` | PASS: servicios iniciados para esta comprobación detenidos; volúmenes conservados. |
| `git diff --check` | PASS en `stir-frontend` y `stir-doc`. |

El fixture no valida el procesamiento funcional de una oferta en STIR/osTRIS. El endpoint y el payload de oferta se contrastaron con el código backend (`OfferRequest`, `NegotiationService`); no se cambió ese contrato.

## Pendientes identificados tras el primer incremento

- Publicar/taggear el artefacto y declarar una matriz de versiones Shell/STIR probada. Por ahora se genera desde una revisión Git fijada.
- Ensayar actualización entre dos revisiones publicadas preservando la segunda presentación. El segundo incremento de abajo aporta un ensayo entre dos commits locales, todavía sin releases.
- Ejecutar el recorrido completo con datos sintéticos en un tenant de prueba dedicado y dos participantes ordinarios. Completado en el segundo incremento de abajo.
- Extraer más capacidades de frontend según las necesidades reales de FreeFolk; este contrato se limita al catálogo.

No se afirma que FreeFolk esté listo para un piloto real ni que Identity Integrity, Ordinary Governance, Consent/Retention, fuentes de referencia o hardware custody estén cerrados.

## Segundo incremento: recorrido funcional y actualización (2026-09-26)

Se usaron worktrees separados para frontend, documentación, composición y un backend **detached en `381b156`**. El checkout normal de `stir-backend` tenía cambios sin confirmar de Ordinary Governance de Claude Code; no se modificó ni se compiló. El override local de Compose apuntó tanto el build backend como sus migraciones a ese worktree estable. `stir-main` y sus vendors permanecieron sin cambios en el checkout normal.

La primera revisión frontend fue `6f296e5` (contrato de catálogo), la segunda `0b3d9af` (métodos aditivos `photos` y `contentUrl`, miniaturas en la segunda presentación y galería STIR consumiendo `photos`). No se cambió `CATALOG_CONTRACT_VERSION = 1`, ya que las entradas anteriores siguen disponibles.

| Comprobación | Resultado |
|---|---|
| `npm test`, build e i18n en la segunda revisión | PASS: 39 pruebas, build del ESM y ambas extensiones, 12 locales/438 claves. |
| Compose con backend estable y segunda presentación | PASS: migraciones, Shell, STIR, osTRIS y proxy saludables. |
| Playwright + API real en tenant nuevo de pruebas | PASS: Ana publicó un anuncio y subió un PNG sintético; Pedro, con una sesión ordinaria, abrió la segunda presentación, vio anuncio/foto, envió una oferta real y STIR persistió negociación, autor y anuncio correctos. La galería STIR también leyó la foto. |
| Ensayo de actualización `6f296e5` → `0b3d9af` | PASS: se creó anuncio/foto/negociación con la revisión anterior, se reconstruyó **sólo `stir-ui`** con la segunda revisión y el mismo Pedro pudo volver a leer el anuncio, ver la foto en ambas presentaciones y abrir el hilo anterior. Backend, Shell, osTRIS y volúmenes se mantuvieron. |
| Cierre y datos sensibles | PASS: Compose `down` sin borrar volúmenes; fichero temporal local con la contraseña de prueba eliminado. |

Las escrituras fueron exclusivamente en tenants nuevos `catalog-e2e-*` de desarrollo. Los scripts reproducibles están en `stir-main/scripts/community_catalog_browser_e2e.py` y `community_catalog_upgrade_e2e.py`; el manifest y proxy opcionales están en `stir-main/examples/community-catalog/`. En este ensayo se usó un override `.local` adicional para apuntar a los worktrees aislados.

Esto prueba una actualización entre **dos commits locales fijados**, no entre dos releases publicadas. Quedan pendientes un tag/release del contrato, una matriz publicada de compatibilidad Shell/STIR, y la extracción de otros recorridos que FreeFolk decida personalizar. La oferta real comprobada crea una negociación STIR; no se probó aquí un commit económico osTRIS.
