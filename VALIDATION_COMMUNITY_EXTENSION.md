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

## Pendiente para cerrar la fase 1

- Publicar/taggear el artefacto y declarar una matriz de versiones Shell/STIR probada. Por ahora se genera desde una revisión Git fijada.
- Ensayar actualización entre dos revisiones publicadas preservando la segunda presentación; sólo hay una revisión de esta entrada pública.
- Ejecutar el recorrido completo con datos sintéticos en un tenant de prueba dedicado y dos participantes ordinarios, si el alcance del siguiente incremento necesita comprobar la oferta funcional y su recepción en STIR. Esta validación mantuvo la instancia sin escrituras de prueba.
- Extraer más capacidades de frontend según las necesidades reales de FreeFolk; este contrato se limita al catálogo.

No se afirma que FreeFolk esté listo para un piloto real ni que Identity Integrity, Ordinary Governance, Consent/Retention, fuentes de referencia o hardware custody estén cerrados.
