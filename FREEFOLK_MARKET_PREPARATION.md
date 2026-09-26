# FreeFolk Market — preparación sobre STIR

Estado: propuesta, 2026-09-26. Depende de [la guía genérica de extensión](COMMUNITY_EXTENSION_GUIDE.md). No crea todavía un módulo, un fork, una instancia pública ni una nueva autoridad económica.

Primer incremento técnico en marcha: STIR dispone ahora de una entrada ESM de catálogo y una segunda presentación de ejemplo que la consume; véase `stir-frontend/examples/community-catalog/README.md`. Ya se comprobó la carga en Shell y lectura real del catálogo. Faltan una oferta funcional real en tenant de prueba, una release fijada y el ensayo de upgrade entre revisiones. La fase 1 no se declara cerrada.

La [validación ejecutada](VALIDATION_COMMUNITY_EXTENSION.md) ya incluye login/consulta reales y un recorrido de detalle/oferta con fixture de navegador. Continúan pendientes una oferta real en tenant de prueba, release fijada y ensayo de actualización.

## Qué podemos comenzar

Podemos preparar la distribución, definir su dominio específico y demostrar reutilización del frontend antes de terminar todas las capacidades pendientes de STIR. La disponibilidad para producción es otra decisión: no se deduce de que exista un marketplace funcional o un despliegue Docker.

Propuesta de composición:

```text
Frontend FreeFolk (presentación propia + capacidades STIR reutilizadas)
    ├── Shell/Core: sesión, tenant, permisos y contexto
    ├── API STIR: catálogo, negociación, acuerdo, recorridos comunitarios
    │     └── osTRIS: autoridad económica → Ledger según composición
    └── API módulo FreeFolk: sólo dominio específico

Composición FreeFolk: versiones fijadas + configuración + datos/secretos propios
```

El módulo FreeFolk no es obligatorio para cambiar de marca. Se creará cuando haya una primera regla o entidad propia confirmada. No inventar comisiones, logística, reputación, afiliación, pagos o políticas comunitarias para justificarlo. El frontend propio sí es un objetivo explícito.

## Preparación y dependencias

| Frente STIR | Evidencia/estado para este diseño | Efecto sobre FreeFolk |
|---|---|---|
| Marketplace y EXCHANGE | Implementados según código y documentación local; no revalidados en runtime en esta tarea | Permiten diseñar el primer recorrido reutilizable. |
| Identity Integrity | Ya hay integración parcial de continuidad y proyección; persisten límites descritos en `PARTICIPANT_INDEPENDENCE.md` | Reutilizar estado/aseguramiento real; nunca anunciar independencia probada por contar cuentas. |
| Ordinary Governance | Pendiente según el plan comunicado; Seven Keys existente no demuestra que esté terminado | Mantener implementación en STIR; integrar cuando haya contratos y evidencia. |
| Consent/Retention | Pendiente según el plan comunicado; no se certifica su cobertura aquí | Definir datos propios y dependencias antes de recogerlos en un piloto real. |
| Fuentes LISTING/WANTED/SEED | Trabajo pendiente según el plan; un anuncio actual `WANTED` no demuestra una fuente de evidencia implementada | No convertir anuncios en observaciones o referencias por cuenta de FreeFolk. |
| WebAuthn / hardware custody | No se da por implementado; WebCrypto actual no equivale a custodia hardware | Mantener firma común; no diseñar un signer alternativo ni asumir compatibilidad criptográfica. |
| Extensibilidad frontend | Bundle único, rutas incrustadas, sin API pública granular | Es el trabajo habilitador inmediato para una UI propia actualizable. |

Esta tabla es una dependencia de planificación, no una auditoría completa del roadmap ni del trabajo remoto activo de Claude Code.

## Tres fases

### 1. Demostrar que STIR se puede reutilizar

En STIR, implementar el primer recorrido descrito en la guía y una segunda presentación de prueba. Publicar entradas públicas, límites y matriz compatible. Mantener la UI STIR consumiendo el mismo contrato. Esto puede avanzar sin resolver los SPEC GAP económicos o constitucionales.

Salida verificable: actualizar una revisión de STIR y demostrar que la segunda presentación conserva su aspecto y recibe el comportamiento común sin copiar código. Una guía o una maqueta por sí solas no completan esta fase.

### 2. Crear la distribución FreeFolk y su primer dominio propio

Definir antes de generar: comunidad destinataria, primer recorrido distintivo, datos propios, relación con recursos STIR, permisos y alcance del piloto. Son decisiones de producto todavía abiertas, no bloqueos para la fase 1.

Crear la composición y frontend FreeFolk con pins de STIR y upstreams. Para el módulo opcional usar el DevKit vigente y su manifiesto, no un scaffold manual. El workspace será el repositorio contenedor y excluirá los repositorios hijos independientes; no anidar otro workspace dentro. Resolver nombres definitivos con el scaffolder antes de crear carpetas.

El módulo tendrá API, esquema, historial Flyway, permisos e i18n propios. Las extensiones puramente visuales no obligan a crear backend. Reutilizar autenticación/contexto Shell y respetar las restricciones de publicación de Core binario. El repositorio de composición FreeFolk será dueño de su configuración de despliegue; las mejoras genéricas de composición deben poder volver a STIR sin arrastrar marca o reglas FreeFolk.

Salida verificable: una instalación independiente construida con dependencias públicas y un recorrido completo de valor específico, con pruebas del módulo y `validate-idax-module --full` cuando exista módulo.

### 3. Habilitar un piloto conforme a las capacidades realmente disponibles

Revisar con evidencia qué versiones incluyen Identity Integrity, Ordinary Governance, Consent/Retention, fuentes de referencia y custodia, y cuáles necesita el alcance concreto del piloto. No exigir por nombre toda una lista para cualquier prototipo, ni activar operaciones que requieran garantías ausentes.

Separar pruebas internas con datos sintéticos de actividad real. Antes de abrir el piloto: validación de aislamiento y autoridades, recorrido de consentimiento/retención aplicable, ceremonia/custodia apropiada, observabilidad, backups/restauración y ensayo de actualización. La topología productiva preparada en STIR no demuestra que esto se haya ejecutado para FreeFolk.

## Coordinación con la evolución de STIR

Esta propuesta no reordena el trabajo activo de Claude Code. Acordar el punto de extracción frontend antes de modificar los mismos archivos. Cada nueva capacidad genérica permanece en STIR; FreeFolk incorpora su release cuando su matriz y pruebas la admitan.

En futuros incrementos STIR, registrar: contrato consumible, permisos/autoridad, estado funcional y errores, traducciones, recorrido UI común, dependencias de versión, migración y pruebas de conformidad. Evitar un segundo framework antes de que la primera extracción pruebe que hace falta.

No se ha enviado ningún mensaje a Claude Code ni creado una tarea remota. Estos documentos son la base revisable para coordinar ese siguiente incremento.

## Evidencia de esta preparación

Cambios exclusivamente en el repositorio `stir-doc`: esta hoja de preparación, la guía genérica y su enlace desde README. Inspeccionados código frontend/API, branding backend, cliente osTRIS, manifest/proxy/pins de composición y contrato DevKit.

Sin cambios de generador, regeneración, migraciones, contratos runtime o repos Java/.NET. Sin builds ni ejecución de servicios: no se afirma compatibilidad operativa nueva. La validación de esta entrega se limita a revisión documental, enlaces locales y whitespace; la implementación y el ensayo real de upgrade siguen pendientes.
