# Validation: primer consumidor comunitario del catálogo

## Mixed Consideration Extension Proof (2026-09-26)

Ramas aisladas: `stir-backend: codex/mixed-consideration-contract`, `stir-doc: codex/mixed-consideration-docs` y repositorio experimental `ffm-mixed-proof: codex/mixed-consideration-proof`. El checkout activo de Claude Code no se ha modificado.

La revisión genérica A de STIR es `223f294`: Offer y Agreement guardan namespace/digest externos opacos, con migración PostgreSQL V13 **provisional**. La revisión B es `43dfd2e`: añade el mismo compromiso a la vista de detalle del snapshot, sin alterar el contrato previo. El módulo FFM experimental se sirve por Shell desde otro repositorio y otra imagen; STIR y osTRIS no importan ese módulo. La imagen FFM final usada en el ensayo A→B fue `sha256:18526915b89478adb5197a1d07dcce0552634e50db06a68a993f839288ac0add` y no se reconstruyó entre A y B.

| Comprobación de Mixed Consideration | Resultado |
|---|---|
| `mvn test -q` en STIR A y `mvn -q -Dtest=ExternalContractCommitmentTest test` en B | PASS; contrato genérico y snapshot. |
| `npm test` en módulo FFM | PASS: 7 pruebas; tres modalidades, contraoferta/digest, fees separados, resultados FIAT y `SETTLEMENT` osTRIS independiente. El adaptador unitario no sustituye la prueba real. |
| Compose aislado `ffm-proof` con STIR B | PASS: tres listings, negociaciones y Agreements reales en tenant sintético; FIAT 70 EUR/fee 3,50 EUR simulados por `TEST_PAYMENT_PROVIDER`; EXCHANGE real de 30 unidades y SETTLEMENT real separado de 1,50 unidades en osTRIS, ambos firmados y `COMMITTED`. La comisión no reduce el precio original. |
| Aislamiento | PASS: acceso cruzado desde otro tenant rechazado por API STIR/FFM; la cuenta de plataforma se registró con permiso de gestión del tenant y las firmas osTRIS provinieron de claves distintas. |
| Frontend propio en Shell | PASS: Playwright/Edge mostró el Agreement mixto real, desglose de precio y fees, y estados de ejecución en una UI externa de diseño propio. |
| Actualización STIR A→B con datos persistentes | PASS: los mismos listings, negociaciones, Agreements, estado FIAT, cuenta de pago sintética, EXCHANGE y SETTLEMENT osTRIS se releyeron antes y después de sustituir sólo la imagen STIR. No se reconstruyó FFM ni se migró manualmente un fork. Ambas revisiones son commits locales, no releases publicadas. |

La prueba utiliza `scripts/e2e.py setup|verify` y `scripts/browser.py` del repositorio experimental `stir/.local/ffm-mixed-proof`; el proyecto Compose y sus volúmenes son exclusivos de la prueba. Se usó el puerto publicado `18096` para STIR del ensayo porque `8096` estaba ocupado por otra instancia, que no se detuvo. Los datos son sintéticos y el PSP sólo simula evidencias: no hay dinero FIAT real, callback firmado, vault, RLS SQL ni garantías de transacción/idempotencia productivas en la extensión. Tampoco hay valoración fiscal implementada.

Al terminar se eliminó el marcador ignorado que contenía contraseñas temporales y se detuvo `ffm-proof` con `docker compose down` sin `-v`; los volúmenes de ensayo quedaron preservados. La instancia `stir-dev` que ocupaba `8096` siguió funcionando y no fue modificada.

La corrección arquitectónica de no convertibilidad figura en [Mixed Consideration](MIXED_CONSIDERATION_EXTENSION.md). No hay campos de paridad ni conversión. La valoración fiscal por operación permanece como `LEGAL/TAX SPEC GAP`; una eventual valoración DAC7 no debe presentarse como renta imponible del vendedor ni impuesto debido.

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

## Community Extension Integration MVP (2026-09-26)

Fase de inspección e integración prevista tras cerrar Ordinary Governance (`stir-backend 470e2db`, `stir-frontend 28ca133`, `stir-main 8bbac8f`, `stir-doc 87b495f`, `stir-workspace 05dc152`). Objetivo: subir a `main` únicamente las abstracciones genéricas ya demostradas por los incrementos anteriores de este documento, no FreeFolk Market ni Mixed Consideration en sí.

### Inspección

Localizadas todas las ramas Codex/Claude locales relacionadas (ninguna estaba en `origin` de ningún repo). Aplicables: `stir-backend codex/mixed-consideration-contract` (`223f294`,`43dfd2e`), `stir-frontend codex/community-catalog-next` (`6f296e5`,`0b3d9af`), `stir-main codex/community-catalog-e2e` (`b4b28fb`), `stir-doc codex/mixed-consideration-docs` (`837501b`,`7b4ffbb`,`0bddd5b`). Obsoletas: `codex/community-value-references` y `codex/stir-foundation` en los cuatro repos (snapshots muy anteriores, `main` ya muy por delante), `codex/community-value-references-e2e` en `stir-main` (ya fusionada, diff vacío), `claude/community-catalog-consumer-mvp` y `codex/community-extension-next`/`claude/community-extension-guide-mvp` en `stir-doc` (ancestros de las ramas anteriores). Sin conflictos de migración (`V13` seguía libre) ni de seguridad/autorización. Único SPEC GAP real encontrado: `purpose=SETTLEMENT` no está respaldado por `OSTRIS_INTEGRATION.md` - ver [Mixed Consideration](MIXED_CONSIDERATION_EXTENSION.md).

### Integración

Cada rama aplicable se rebasó (cherry-pick, sin conflictos reales) sobre el `main` actual, no sobre su baseline original. Cambios añadidos sobre lo ya descrito arriba: `offerPayload()` reenvía `externalContractNamespace`/`externalContractDigest` cuando una distribución los provee (`stir-frontend`); `GET /api/stir/instance` expone `stirVersion`/`catalogContractVersion`/`externalContractSchemaVersion` (`stir-backend`, `InstanceController`); el rehearsal de upgrade en `stir-main` compara esos tres campos antes/después. `COMMUNITY_EXTENSION_HANDOFF.md` se retiró: era una nota de coordinación puntual dirigida a Claude Code, ya cumplida por esta integración.

| Comprobación | Resultado |
|---|---|
| `mvn -q verify` en `stir-backend` (cherry-pick + compatibilidad) | PASS: migraciones V1-V13 aplicadas en 5 corridas Testcontainers independientes, incluidas `ExternalContractCommitmentTest` e `InstanceControllerTest` (nueva). |
| `npm test`/`build`/`i18n:validate` en `stir-frontend` (cherry-pick + passthrough) | PASS: 40 pruebas, 12 locales/490 claves, build ESM+ambas extensiones. |
| `docker compose up -d --build` con backend integrado y frontend **anterior a esta integración** (`main` sin catálogo) | PASS: pila completa saludable; confirma que el backend integrado no rompe el frontend previamente publicado. |
| Fixture real (listing, foto auténtica, negociación) creado por HTTP contra ese frontend anterior | PASS: sin la extensión de catálogo, usando sólo la API estándar de STIR. |
| **Upgrade real de capacidad**: reconstruir sólo `stir-ui` a la revisión integrada (con `catalog-client.js` + segunda presentación) manteniendo backend/datos | PASS (a): la UI estándar de STIR sigue mostrando la foto y la negociación creadas antes de que el catálogo existiera. PASS (b): la segunda presentación, inexistente antes del upgrade, lee ese mismo anuncio y foto sin ninguna migración de datos. Prueba directa del criterio de éxito: "actualizar STIR no requiere migrar manualmente un fork downstream". |
| `scripts/community_catalog_browser_e2e.py` y `community_catalog_upgrade_e2e.py` (tal como quedaron committeados) | PASS, incluida la nueva comparación de `stirVersion`/`catalogContractVersion`/`externalContractSchemaVersion` capturada en el marcador. |
| Regresión: `scripts/smoke.py`, `marketplace_e2e.py`, `economic_exchange_e2e.py`, `ordinary_governance_e2e.py` | PASS: sin regresión en listados, negociación/oferta ni gobernanza ordinaria tras los cambios de `Offer`/`OfferRequest`/`CounterRequest`/`api.js`. |
| Cierre | Compose `down` sin borrar volúmenes; marcadores temporales locales eliminados; ningún secreto impreso. |

No se ejecutó aquí una matriz completa de los ocho scripts E2E preexistentes (Seven Keys, Market Integrity, Participant Independence, Community Value Governance): la regresión se acotó a las áreas realmente tocadas (negociación/oferta, referencia/gobernanza, instancia pública). Queda como comprobación pendiente antes de una release pública ejecutar la matriz completa.

### Qué subió y qué no

Subió: el compromiso opaco Offer→Agreement (`V13`, sin FIAT/fee/fiscal); el contrato `catalog-client.js` con STIR como primer consumidor y su passthrough opcional del compromiso opaco; el ejemplo de segunda presentación y su composición opt-in; los E2E de catálogo y el rehearsal de upgrade; la declaración mínima de compatibilidad en `GET /api/stir/instance`. No subió: nada de `.local/ffm-mixed-proof` (FIAT, PSP, comisiones, valoración fiscal, `purpose=SETTLEMENT`); ningún framework de plugins ni registro de capacidades más allá de los tres campos de versión; ninguna credencial SQL o de servicio privilegiada expuesta a extensiones; `COMMUNITY_EXTENSION_HANDOFF.md` (nota de coordinación ya cumplida, no documentación permanente).
