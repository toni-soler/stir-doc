# Extender STIR para comunidades e instancias independientes

El [experimento de contraprestación mixta](MIXED_CONSIDERATION_EXTENSION.md) probó el límite de esta guía con una obligación FIAT externa y una osTRIS existente. Distingue el contrato genérico incorporado a STIR del dominio específico que permanece en la distribución, así como las pruebas y SPEC GAP aún pendientes.

Para dominios económicos externos, el único contrato upstream es un **compromiso opaco** por Offer (`externalContractNamespace` + `externalContractDigest`) congelado en el snapshot del Agreement (`schemaVersion=3`). La extensión conserva y verifica sus términos inmutables, ejecuta sólo por APIs autenticadas bajo el actor/tenant y almacena su propio estado; el compromiso no concede permiso para escribir tablas STIR/osTRIS ni declara atomicidad entre proveedores. STIR no interpreta FIAT, comisiones ni políticas de distribución. La valoración fiscal futura, si procede, será metadato por operación y finalidad legal, sin paridad ni convertibilidad de unidades osTRIS; ninguna cifra DAC7 se presentará como renta imponible o impuesto debido del vendedor.

Estado: **incorporado a `main`**, 2026-09-26 (Community Extension Integration MVP), tras un incremento previo de preparación documental y prueba en ramas locales aisladas. No declara un SDK de componentes publicado ni un framework de plugins; ver "Contrato frontend" más abajo para lo que existe realmente hoy. Primer consumidor previsto: [FreeFolk Market](FREEFOLK_MARKET_PREPARATION.md), que sigue sin dominio propio.

**Contrato de catálogo (integrado):** `stir-frontend` expone una entrada pública de catálogo en `src/catalog-client.js`, compilada como `dist/community/stir-catalog.mjs`, y una segunda presentación de ejemplo en `examples/community-catalog/`. STIR's own UI (`listing.jsx`, `extension.jsx`) consume ese mismo `createCatalogClient` - es su propio primer consumidor, no una copia paralela. El contrato cubre catálogo, fotos, oferta y un passthrough opcional del compromiso opaco de contraprestación externa (`offerPayload`). `mvn verify`, `npm test`/`build`/`i18n:validate` y los E2E HTTP/navegador enumerados en [la validación](VALIDATION_COMMUNITY_EXTENSION.md) pasan sobre esta integración. Sigue faltando un SDK de componentes, una release etiquetada y una matriz de versiones publicada más allá del campo `stirVersion` descrito abajo.

**Compromiso opaco (integrado):** `Offer`/`Agreement` llevan `externalContractNamespace`/`externalContractDigest` opcionales (migración `V13`, ambos NULL por defecto, ambos obligatorios juntos o ninguno). No hay tabla, import ni referencia a FreeFolk en STIR/osTRIS.

**Compatibilidad (integrado, mínimo):** `GET /api/stir/instance` (público, sin autenticar) expone `stirVersion`, `catalogContractVersion` y `externalContractSchemaVersion` - las únicas versiones que STIR realmente hace cumplir hoy. No es un endpoint de capacidades ni promete compatibilidad ilimitada; ver la sección de versionado.

## Objetivo y decisión propuesta

Una comunidad debe poder combinar versiones publicadas de STIR, osTRIS, IDAX Ledger y Shell con su módulo de dominio y su propia experiencia visual. STIR conserva un producto de referencia completo. Las distribuciones incorporan sus mejoras actualizando dependencias verificadas, sin mantener una copia divergente de STIR.

La unidad de reutilización inicial es el **servicio STIR y sus APIs existentes**. La reutilización granular del frontend necesita un contrato público adicional. No se propone convertir ahora el backend en una biblioteca embebida, crear un sistema de plugins de servidor ni migrar STIR entero al DevKit.

Separar tres conceptos:

- **Instancia**: despliegue independiente, con datos, secretos, operadores y ciclo de actualización propios.
- **Comunidad**: autoridad económica y de gobierno explícitamente vinculada dentro de una instancia. Un tenant no demuestra por sí solo esa vinculación.
- **Distribución**: composición versionada de software, módulos, configuración y frontend. FreeFolk sería una distribución que puede desplegarse en instancias independientes.

No se establece federación ni se comparten automáticamente usuarios, saldos, identidades, claves o autoridades entre instancias.

## Baseline inspeccionada e integrada

Revisión que cerró el Community Extension Integration MVP, 2026-09-26 (tras Ordinary Governance en `main`): `stir-doc 87b495f`, `stir-backend 470e2db`, `stir-frontend 28ca133`, `stir-main 8bbac8f`, más los commits de este MVP encima. Los repositorios pueden seguir evolucionando; revalidar antes de extender más. Los paths siguientes son relativos a los repositorios hermanos, no importaciones autorizadas de sus fuentes.

| Evidencia | Existe hoy | Implicación |
|---|---|---|
| `stir-backend/.../config/InstanceController.java` | `GET /api/stir/instance`, configuración `stir.instance.*`, ahora también `stirVersion`/`catalogContractVersion`/`externalContractSchemaVersion` | Marca básica sin fork, más una declaración de versión mínima y real; no constituye un sistema de temas completo ni un registro de capacidades. |
| `stir-frontend/src/catalog-client.js`, `scripts/build.mjs` | ESM sin Shell/React/router, `CATALOG_CONTRACT_VERSION=1`; STIR's own `listing.jsx`/`extension.jsx` lo consumen vía `createCatalogClient` | Primer contrato frontend reutilizable real, con STIR como primer consumidor; sigue sin paquete de componentes ni SDK publicado. |
| `stir-frontend/examples/community-catalog/` | Segunda presentación de ejemplo (ruta y marca propias) que importa `dist/community/stir-catalog.mjs` | Prueba la reutilización sin copiar fuentes internas; no es la UI de FreeFolk. |
| `stir-backend/.../negotiation/Offer.java`, `V13__external_contract_commitment.sql` | `externalContractNamespace`/`externalContractDigest` opcionales, ambos o ninguno, en el snapshot `schemaVersion=3` | El único contrato upstream para dominios económicos externos; STIR no interpreta su contenido. |
| `stir-main/examples/community-catalog/`, `scripts/community_catalog_*_e2e.py` | Composición opt-in de ejemplo + E2E HTTP/navegador + rehearsal de upgrade que compara el campo de compatibilidad antes/después | No cambia el manifest ni el proxy por defecto de STIR. |
| `stir-main/deploy/extensions.json` | Manifest de Shell con extensiones STIR/osTRIS/Ledger | Host genérico disponible; no resuelve automáticamente composición interna de páginas STIR. |
| `stir-main/upstream.lock.json` | Pins Git de Core runtime, Shell, Ledger y osTRIS | Patrón aprovechable; el lock actual no incorpora FreeFolk ni fija los propios repos STIR. |
| `stir-main/deploy/Caddyfile` | Proxy por namespaces de API y assets | Punto de composición de servicios sin copiar sus controladores. |
| `stir-backend/.../economic/OstrisClient.java` | HTTP con bearer del usuario y tenant; sin SQL compartido; sólo envía `purpose=EXCHANGE` | Mantener las autoridades y los límites existentes; ver el SPEC GAP de `purpose` en [Mixed Consideration](MIXED_CONSIDERATION_EXTENSION.md). |

Leer también [Architecture](ARCHITECTURE.md), [Shell compatibility](SHELL_COMPATIBILITY.md), [Public boundary](PUBLIC_SOFTWARE_BOUNDARY.md), [Participant independence](PARTICIPANT_INDEPENDENCE.md) y [Governance threat model](GOVERNANCE_CAPTURE_THREAT_MODEL.md). Algunos párrafos históricos de Architecture/Readiness describen etapas anteriores: no inferir de ellos el estado de una capacidad nueva sin inspeccionar código y validación específicos.

## Propiedad de cada capa

| Capa | Responsabilidad | Límite de extensión |
|---|---|---|
| Core / Shell | Sesión, tenant, permisos de plataforma, host, HTTP autenticado, locale | Reutilizar contratos públicos; no duplicar autenticación en cada módulo. |
| osTRIS | Autoridad del protocolo económico, autorización y commit | STIR/FreeFolk no reinterpretan finalización ni fabrican firmas. |
| IDAX Ledger | Evidencia según contratos del stack | No convertir una prueba en autorización comercial o de gobierno. |
| STIR | Anuncios, negociación, acuerdos, integración económica y capacidades comunitarias | Mejoras genéricas aquí; APIs y UX reutilizable con contratos explícitos. |
| Módulo comunitario | Reglas y datos específicos de la comunidad | Esquema, migraciones, permisos y API propios; referencias a IDs STIR mediante contratos públicos. |
| Frontend de distribución | Identidad visual, navegación, composición de recorridos | Reutilizar comportamiento común; personalización visual no concede autoridad. |
| Composición de distribución | Versiones, proxy, configuración, despliegue y actualización | No parchear fuentes vendorizadas para cambiar reglas de producto. |

El módulo no escribe tablas STIR/osTRIS/Ledger ni importa sus repositorios JPA. Una referencia a un anuncio no demuestra acceso: verificar tenant, usuario y recurso con el servicio propietario. Preferir no imponer foreign keys entre esquemas de servicios independientes.

Para un recorrido que modifica varios servicios no existe transacción distribuida implícita. Antes de implementarlo, especificar idempotencia, reintento, estado parcial y reconciliación; si hace falta un evento de STIR, incorporarlo primero como contrato probado en STIR. No asumir que ya existe un event bus, un webhook o una credencial de servicio utilizable.

Los nuevos módulos IDAX siguen `idax-module-devkit/docs/IDAX_NEW_MODULE_CONTRACT.md`: scaffold con `gen-idax-module`, manifiesto `.idax-module.yml`, generador propio para CRUD declarativo, separación generated/custom y `validate-idax-module --full`. El DevKit es la autoridad sobre nombres y estructura. No copiar scaffolding de un módulo antiguo ni exigir código privado de Core. STIR preexistente no se migra por el mero hecho de añadir una distribución.

## Contrato frontend que falta construir en STIR

Implementar por extracción incremental, manteniendo STIR como primer consumidor del mismo contrato que usará una comunidad. El primer nombre concreto es `createCatalogClient(sdk, tenantId)` en el artefacto ESM de catálogo; aún no hay un paquete npm publicado.

1. **Cliente de API público**: primer vertical implementado para catálogo, lectura, escritura y oferta. Transporte inyectado desde Shell, tenant explícito, errores documentados y cancelación en list/read. La entrada externa es el ESM generado, no `src/api.js`.
2. **Comportamiento reutilizable**: operaciones y estado de listado/negociación/acuerdo, sin navegación ni marca incrustadas. Extraer sólo lo que use una segunda presentación real.
3. **Componentes/recorridos reutilizables**: entradas públicas pequeñas, props documentadas, estados vacíos/error/carga y traducciones. Mantener dispositivos, firma y gobierno en recorridos comunes hasta disponer de sustituciones verificadas.
4. **Adaptador de presentación**: rutas y enlaces suministrados por el host, tokens visuales y puntos de composición limitados. Sustituir `/stir` incrustado exige revisar enlaces, notificaciones, redirecciones y deep links, no sólo el router principal.
5. **Registro explícito de capacidades**: cada release describe ID, versión, requisitos de API/host, rutas, permisos, traducciones, componente predeterminado y posibilidad de sustitución. Sigue siendo un contrato propuesto y no un endpoint de capabilities completo: lo único que existe hoy es la declaración mínima de versión en `GET /api/stir/instance` (`stirVersion`, `catalogContractVersion`, `externalContractSchemaVersion`), pensada para que una distribución compruebe compatibilidad antes/después de su propia actualización, no para descubrir rutas, permisos o componentes.

La entrada de biblioteca no registrará módulos en `window`, rutas o traducciones por efectos secundarios al importarse. El adaptador Shell realizará ese registro. React, router e i18n seguirán procediendo del host compatible, evitando una segunda copia de React. CSS acotado y traducciones con namespace, conservando cobertura de los 12 locales.

Cada distribución declara explícitamente qué capacidades usa con UI común y cuáles sustituye. Una capacidad opcional nueva llega como disponible para integrar, no se activa sin revisión. Una corrección de seguridad o requisito obligatorio incompatible debe bloquear la release hasta actualizar el recorrido correspondiente. Desconocer una capacidad no debe conceder permisos ni presentar una operación como completada.

**Límite de la promesa:** un frontend completamente personalizado puede reutilizar lógica y recorridos comunes, pero una pantalla reescrita por completo no hereda automáticamente cambios de interfaz. Cada sustitución tendrá propietario y pruebas de conformidad. Cuanto mayor sea la reutilización, menor será el coste de actualización.

Mientras este contrato no exista, una distribución puede alojar su UI personalizada junto a las pantallas STIR originales bajo `/stir`, enlazando los recorridos que todavía no sustituye. Esto permite adopción incremental; no debe presentarse como la experiencia personalizada final.

## Versionado y actualización

Mantener en la composición un lock completo: repos STIR, módulo comunitario, frontend, Core/Shell/osTRIS/Ledger, patches revisados y artefactos reproducibles. Fijar commits inmutables y, cuando se produzcan imágenes, digests. No usar `latest`, branches móviles ni dependencias `file:` a checkouts del autor.

Versionar por separado API, paquete frontend, contrato de host y esquema de base de datos. Publicar una matriz de combinaciones realmente probadas; una etiqueta SemVer no demuestra por sí sola compatibilidad. Antes del primer contrato público, declarar cuáles superficies son experimentales y cuáles se soportan.

Proceso de upgrade:

1. Leer cambios, deprecaciones, requisitos y notas de migración de cada dependencia; elegir una combinación soportada.
2. Cambiar pins en una rama de la distribución. No hacer merge de toda la historia STIR en el producto.
3. Construir desde checkout limpio y ejecutar contratos del frontend común y de cada sustitución.
4. Ensayar instalación vacía y actualización con datos de la versión anterior, incluyendo aislamiento de tenants y permisos.
5. Probar backup/restauración y decidir si la versión antigua tolera el nuevo esquema. No prometer rollback binario tras cualquier migración: si no es compatible, usar restauración coordinada o corrección hacia delante, considerando escrituras posteriores.
6. Validar E2E en una instancia de prueba y publicar matriz, evidencia y release notes antes de desplegar.

Las migraciones pertenecen al servicio propietario y tienen historial/credenciales separados. La distribución no renumera ni modifica migraciones STIR. Cambios de datos destructivos requieren su propio plan, no una actualización automática de dependencias.

## Seguridad e independencia

Una instancia separada utiliza su propia base de datos/volúmenes, almacenamiento de objetos, secretos, emisores/validadores de identidad según el stack y configuración económica explícita. No reutilizar credenciales de `stir.es` ni copiar bindings/autoridades de otra instancia. Compartir software no implica compartir red Ledger ni comunidad osTRIS; documentar la topología seleccionada y comprobarla.

Mantener RLS forzado y comprobaciones de propietario/partes en servicios. Un permiso de plataforma o superusuario no equivale a autoridad comunitaria. El módulo no introduce rutas alternativas para evitar checks de STIR. Preservar umbrales, Guardian no gobernante, firmas sobre bytes canónicos y bloqueos por SPEC GAP, incluido `REPLACE_CONTROLLER`.

La configuración puede cambiar marca y recorridos, no invariantes constitucionales. Identidad desconocida permanece desconocida; Consent/Retention y custodia no se simulan con textos legales o interruptores visuales. Material sensible no se copia a proyecciones comunitarias sin necesidad y contrato de acceso/retención definido.

## Primera entrega implementable y aceptación

El recorrido elegido fue **catálogo, detalle y navegación hacia negociación**: su contrato frontend se extrajo a STIR, STIR lo usa como primer consumidor y una segunda presentación de ejemplo (ruta y marca distintas) lo consume también.

Criterios de aceptación y su evidencia:

- Ambas presentaciones consumen el mismo artefacto público y backend sin copiar fuentes internas: `createCatalogClient` en `catalog-client.js`, usado por `listing.jsx`/`extension.jsx` de STIR y por `examples/community-catalog/extension.jsx`.
- Sesión, cambio de tenant, locale, errores, permisos, enlaces y aislamiento funcionan en ambas: probado en `tests/catalog-client.test.mjs` y en los E2E de `stir-main`.
- Las pruebas detectan un cambio incompatible en una entrada pública o respuesta consumida: `verify-community-build.mjs` valida los exports públicos y el registro Shell en cada build.
- Un upgrade real entre dos revisiones fijadas se completó conservando la personalización, con evidencia: ver "Community Extension Integration MVP (2026-09-26)" en [la validación](VALIDATION_COMMUNITY_EXTENSION.md) - incluye tanto el rehearsal original entre dos commits que ya tenían el catálogo como uno nuevo, más exigente, entre la revisión de `main` sin esta capacidad y la integrada, con datos reales creados antes de que el catálogo existiera y leídos después por ambas presentaciones sin ninguna migración manual.
- `mvn verify` con PostgreSQL/Testcontainers, tests/build/i18n frontend y los E2E HTTP/navegador aplicables pasan; ningún E2E fue sustituido por mocks.
- Reproducible con fuentes públicas, Core binario documentado y secretos locales nuevos: mismo `python scripts/initialize.py` + `docker compose up -d --build` que el resto de STIR.

Esta entrega se declara **cerrada** para el recorrido de catálogo. El contrato frontend más amplio (componentes/recorridos reutilizables más allá de catálogo, adaptador de presentación genérico, registro de capacidades completo) sigue sin construirse - extraer sólo cuando otra distribución real lo necesite, no por generalización especulativa.
