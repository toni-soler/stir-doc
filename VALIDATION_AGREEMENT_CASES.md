# Validación: casos sobre acuerdos

Fecha: 2026-11-18. Diseño: `AGREEMENT_CASES.md`.

Todo se ejecutó en la VM DEV, en el proyecto aislado `stir-dr` (API en `127.0.0.1:38196`, web en `127.0.0.1:38199`), con osTRIS fijado al commit `f00b244` de la rama `claude/dispute-resolution-contract`. Se eliminó con `down -v --rmi local` y se comprobó que no quedaba nada. stir-dev, ffm-dev y ffm-demo no se tocaron.

Ramas probadas: `claude/agreement-disputes-mvp` de stir-backend, stir-frontend, stir-main y stir-doc, desde `main` tras la fusión del cumplimiento de acuerdos.

## Resultados

| Prueba | Resultado |
|---|---|
| osTRIS `ResolutionLifecyclePostgresTest` y catálogo de permisos | 12/12 |
| osTRIS `mvn verify` completo | 270 tests, 0 fallos nuevos (ver abajo) |
| Backend `mvn verify` (PostgreSQL real, Flyway desde vacío hasta V24) | 316/316 |
| `CaseRulesTest` | 4/4 |
| `CasePostgresTest` (como `idax_app`, bajo RLS, osTRIS simulado) | 6/6 |
| Frontend `node --test`, `i18n:validate`, build | 64/64; 12 idiomas × 875 claves; build correcto |
| `agreement_cases_e2e.py` (HTTP, osTRIS real) | 50/50 |
| `agreement_cases_browser.py` (Chromium, en castellano, sin errores de JavaScript) | PASS |
| Regresiones: `agreement_fulfilment_e2e`, `agreement_fulfilment_browser`, `structured_terms_e2e`, `smoke`, `economic_exchange_e2e`, `negotiation_execution_integrity_e2e`, `marketplace_e2e`, `community_references_e2e`, `governed_boundary_e2e`, `community_value_governance_e2e`, `ordinary_governance_e2e` | todas PASS |

En osTRIS, cinco tests fallan igual en `main` sin este cambio y en este entorno: cuatro de `ProtocolProofOutboxPostgresTest` ("SKIP LOCKED/PROCESSING claim must prevent duplicate delivery") y `ProvisioningAndDiscoveryPostgresTest.aPreExistingUnownedControllerIsFailClosedForCredentialManagement`. Se comprobó ejecutándolos sobre `main` en la misma VM. No los introduce este incremento. La VM necesita `-Dtestcontainers.version=1.21.4` para el Docker actual.

## Lo que comprueba el E2E HTTP

- **Quién revisa:**
  - sin revisores no se abre ningún caso;
  - un superusuario de plataforma y un miembro corriente no pueden nombrarlos;
  - la propietaria del tenant sí;
  - la lista es pública;
  - osTRIS tiene a los mismos tres revisores como autoridad de resolución efectiva, con umbral 2.
- **Abrir:**
  - sólo la parte a la que se debía la obligación;
  - nunca sobre el intercambio económico;
  - la otra parte recibe `CASE_OPENED`;
  - nadie más ve el caso, tampoco un superusuario.
- **Pruebas:** la prueba `REVIEWERS_ONLY` del vendedor llega a los revisores; el comprador ve que existe, sin contenido.
- **Contención:**
  - está acotada en el tiempo;
  - pausa una enmienda (`409 CASE_CONTAINMENT_ACTIVE`);
  - no cambia saldo, suelo ni capacidad de nadie;
  - el tablero la muestra.
- **Conclusión y apelación:**
  - sólo el revisor que tomó el caso emite la conclusión;
  - la base de osTRIS queda `PENDING`;
  - ni el tiempo ni un revisor solo la hacen firme;
  - la conducta no se valora antes;
  - la vendedora apela, y la autora no puede decidir la apelación;
  - otro revisor la confirma, la base pasa a `FINAL`, la contención se levanta;
  - la obligación conserva su estado declarado.
- **Conducta:** la atribución a un tercero de buena fe es una decisión aparte, y no admite sanción.
- **Corrección:**
  - es un `REVERSAL` de exactamente ese intercambio, con base `FINAL_DISPUTE_RESOLUTION`;
  - una parte no puede firmarla;
  - el revisor firma el texto exacto, con las entradas negadas y las dos cuentas cubiertas;
  - con una firma de dos no se mueve nada; con la segunda, osTRIS la registra;
  - los saldos vuelven y el `EXCHANGE` original sigue registrado;
  - la cronología cuenta cada paso.
- **Sanción:**
  - una atribución firme de incumplimiento no sanciona por sí sola;
  - la advertencia la decide un revisor expresamente;
  - no cambia historia, saldo ni crédito;
  - se apela y la revoca otro revisor;
  - no hay `PENALTY`, `RESTITUTION` ni `WRITE_OFF` en osTRIS.

## Correcciones durante la validación

| Hallazgo | Corrección |
|---|---|
| La UI y la API de STIR nombraban "FFC", la unidad de FreeFolk Market. STIR no debe llevar semántica de una distribución. | Los campos pasan a `economicCorrection` y `ECONOMIC_REVERSAL`. Los textos hablan de la unidad comunitaria en los 12 idiomas. |
| `ECONOMIC_REVERSAL` no cabía en `varchar(16)`. | `kind` es `varchar(24)`. |
| Un segundo caso sobre la misma obligación, o una segunda corrección, tocaban el índice único y abortaban la transacción. | Se comprueba antes y se responde 409; el índice queda como última defensa ante carreras. |

## No verificado

- El despliegue en stir.es: todavía no tiene este incremento.
- Firmas de revisores con WebAuthn: el navegador usa la clave Ed25519 del dispositivo, igual que en los intercambios.
