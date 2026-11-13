# Validación: integridad de negociación y ejecución económica

Diseño: [NEGOTIATION_EXECUTION_INTEGRITY.md](NEGOTIATION_EXECUTION_INTEGRITY.md). Fecha: 2026-11-13. Todo en la VM DEV, en proyectos Compose propios y aislados (`stir-nei-base`, `stir-nei-fix`, `stir-nei-comb`), retirados con `down -v --rmi local`; ninguno toca `stir-dev` ni otro stack compartido, y ninguno toca stir.es.

## Revisiones

| Repo | Base | Rama `claude/negotiation-execution-integrity-mvp` |
|---|---|---|
| stir-backend | `bc455fa` | `9227d10`, `aceaed2` |
| stir-frontend | `9fb2211` | `b1a0115` |
| stir-main | `659b89f` (E2E ejecutado sobre `cdc87e8`, ver nota) | `2a4cf81` |
| stir-doc | `07b62c0` | este documento |

Nota: `stir-main@main` publicado exige la migración `V20` que sólo existe en la rama `claude/governed-state-effectivity-boundary` de stir-backend; el stack se compone con `stir-main@cdc87e8`, la última revisión que arranca con `stir-backend@main`. Esta rama no añade migraciones.

## Resultados

| Gate | Antes (`bc455fa`) | Después |
|---|---|---|
| `negotiation_execution_integrity_e2e.py` | 2/14 FIXED, 12 REPRODUCED | 16/16 FIXED (dos ejecuciones independientes) |
| `mvn verify` (PostgreSQL/Testcontainers, contenedor `maven:3.9.9-eclipse-temurin-21` en la VM) | — | 243 tests, 0 fallos, 0 errores, 0 omitidos |
| `marketplace_e2e.py` | — | PASS |
| `economic_exchange_e2e.py` (intercambio real con firmas Ed25519, reconciliación, idempotencia, rechazo por límite de crédito) | — | PASS |
| stir-frontend `npm test` (Node 20) | — | 46/46 |
| stir-frontend `i18n:validate` | — | 12 locales, 562 claves |
| Consumidor externo sobre esta rama (E2E HTTP 61/61 y navegador 23/23) | — | PASS |

Tests nuevos o ajustados: `NegotiationServiceTest` (roles por dirección, revisión distinta a la de apertura, partes esperadas, ofertas sin importe, carreras perdidas y restricciones desconocidas), `ListingServiceTest` (cambio de dirección con negociación abierta, bloqueo de fila), `TradeServiceTest` (transacción de otro Agreement con y sin digest reescrito, importe/dirección/unidad/cuenta/transactionId distintos, fila `stir.trade` alterada, camino correcto intacto) y `api.test.mjs` (envío de las partes esperadas).

En los logs del servicio aparecen `duplicate key` de Hibernate durante las carreras: Hibernate los registra antes de que `Conflicts` los traduzca a 409. No son respuestas 5xx.

## No verificado

- La revisión desplegada en stir.es.
- Rendimiento con muchas negociaciones simultáneas sobre un mismo anuncio (el bloqueo compartido serializa sólo frente a la edición del dueño).
