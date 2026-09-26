# Extender STIR para comunidades e instancias independientes

Estado: **propuesta de arquitectura**, 2026-09-26. No declara un SDK nuevo disponible ni autoriza cambios de protocolo. Primer consumidor previsto: [FreeFolk Market](FREEFOLK_MARKET_PREPARATION.md).

**Incremento en curso:** `stir-frontend` contiene una entrada pública de catálogo en `src/catalog-client.js`, compilada como `dist/community/stir-catalog.mjs`, y una segunda presentación de ejemplo en `examples/community-catalog/`. El contrato cubre catálogo, fotos y oferta. Ya se ensayó una actualización entre dos commits; faltan un SDK de componentes, una release etiquetada y la matriz de versiones publicadas.

La [validación del primer consumidor](VALIDATION_COMMUNITY_EXTENSION.md) registra una carga real en Shell y una consulta real a STIR, además de pruebas de UI con fixture para detalle y oferta. El ensayo posterior usa una oferta real y conserva datos al pasar entre dos commits; aún falta repetirlo entre releases publicadas.

El segundo incremento añade fotos autenticadas al mismo cliente y registra una oferta funcional con dos participantes, más un upgrade ensayado entre commits `6f296e5` y `0b3d9af`. El artefacto sigue sin tag/release público ni matriz de versiones publicada. Los detalles y límites están en [la validación](VALIDATION_COMMUNITY_EXTENSION.md).

## Objetivo y decisión propuesta

Una comunidad debe poder combinar versiones publicadas de STIR, osTRIS, IDAX Ledger y Shell con su módulo de dominio y su propia experiencia visual. STIR conserva un producto de referencia completo. Las distribuciones incorporan sus mejoras actualizando dependencias verificadas, sin mantener una copia divergente de STIR.

La unidad de reutilización inicial es el **servicio STIR y sus APIs existentes**. La reutilización granular del frontend necesita un contrato público adicional. No se propone convertir ahora el backend en una biblioteca embebida, crear un sistema de plugins de servidor ni migrar STIR entero al DevKit.

Separar tres conceptos:

- **Instancia**: despliegue independiente, con datos, secretos, operadores y ciclo de actualización propios.
- **Comunidad**: autoridad económica y de gobierno explícitamente vinculada dentro de una instancia. Un tenant no demuestra por sí solo esa vinculación.
- **Distribución**: composición versionada de software, módulos, configuración y frontend. FreeFolk sería una distribución que puede desplegarse en instancias independientes.

No se establece federación ni se comparten automáticamente usuarios, saldos, identidades, claves o autoridades entre instancias.

## Baseline inspeccionada

Revisión local de 2026-09-26: `stir-doc a20bdee`, `stir-backend 381b156`, `stir-frontend 0b980dc`, `stir-main 10875d9`. Los repositorios pueden seguir evolucionando; revalidar antes de implementar. Los paths siguientes son relativos a los repositorios hermanos, no importaciones autorizadas de sus fuentes.

| Evidencia | Existe hoy | Implicación |
|---|---|---|
| `stir-backend/.../config/InstanceController.java` | `GET /api/stir/instance`, configuración `stir.instance.*` | Marca básica sin fork; no constituye un sistema de temas completo. |
| `stir-frontend/src/extension.jsx`, `scripts/build.mjs` | Baseline: un bundle IIFE, SDK global de Shell, registro `stir`, rutas `/stir` incrustadas | La extracción posterior añade un artefacto ESM de catálogo, todavía sin paquete de componentes. |
| `stir-frontend/src/api.js` | Baseline: clientes sobre `sdk.fetchWithAuth`, tenant explícito | La extracción posterior mueve el catálogo a `catalog-client.js`; el resto sigue interno. |
| `stir-main/deploy/extensions.json` | Manifest de Shell con extensiones STIR/osTRIS/Ledger | Host genérico disponible; no resuelve automáticamente composición interna de páginas STIR. |
| `stir-main/upstream.lock.json` | Pins Git de Core runtime, Shell, Ledger y osTRIS | Patrón aprovechable; el lock actual no incorpora FreeFolk ni fija los propios repos STIR. |
| `stir-main/deploy/Caddyfile` | Proxy por namespaces de API y assets | Punto de composición de servicios sin copiar sus controladores. |
| `stir-backend/.../economic/OstrisClient.java` | HTTP con bearer del usuario y tenant; sin SQL compartido | Mantener las autoridades y los límites existentes. |

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
5. **Registro explícito de capacidades**: cada release describe ID, versión, requisitos de API/host, rutas, permisos, traducciones, componente predeterminado y posibilidad de sustitución. Es un contrato propuesto; no existe hoy un endpoint de capabilities que se pueda consumir.

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

Elegir **un recorrido existente**: catálogo, detalle y navegación hacia negociación. Extraer su contrato frontend en STIR, usarlo también en STIR y demostrar una segunda presentación con ruta y marca distintas. No construir primero una plataforma genérica de plugins.

La entrega estará completa cuando:

- Ambas presentaciones consuman el mismo artefacto público y backend sin copiar fuentes internas.
- Sesión, cambio de tenant, locale, errores, permisos, enlaces y aislamiento funcionen en ambas.
- Las pruebas detecten un cambio incompatible en una entrada pública o respuesta consumida.
- Un upgrade real entre dos revisiones fijadas se complete conservando la personalización, con diff y evidencia.
- `mvn verify` con PostgreSQL/Testcontainers, tests/build/i18n frontend y E2E aplicables pasen; no sustituir E2E por mocks.
- Una persona o IA pueda reproducirlo con fuentes públicas, Core binario documentado y secretos locales nuevos.

Esta guía es preparación documental: ninguno de esos criterios de implementación se declara cumplido aquí.
