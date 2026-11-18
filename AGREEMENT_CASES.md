# Casos sobre acuerdos: revisar el efecto sin juzgar a la persona

Estado: **implementado** en las ramas `claude/agreement-disputes-mvp` de stir-backend, stir-frontend, stir-main y stir-doc, y `claude/dispute-resolution-contract` de osTRIS (2026-11-18). La validación está en `VALIDATION_AGREEMENT_CASES.md`.

`AGREEMENT_FULFILMENT.md` deja una obligación `IN_DISAGREEMENT` cuando una parte dice "no recibido" y la otra dice que cumplió. STIR no decide quién tiene razón. Este incremento añade el camino para revisarlo: un caso que examina **lo que pasó con la obligación**, no a la persona, y que puede terminar en una reparación del efecto.

Todo es genérico. Ninguna distribución aporta código ni semántica. La corrección económica se hace en la unidad comunitaria de osTRIS; una distribución con su propio tramo externo (el EUR de FreeFolk Market) lo corrige en su propio sistema, citando el caso si quiere.

## Cinco objetos separados

El libro pide que la disputa, la reparación del efecto, la atribución de conducta, la sanción y la apelación no se confundan. Aquí son objetos distintos, cada uno con su autor, su estado y su apelación:

| Objeto | Qué responde | Quién |
|---|---|---|
| **Caso** (`agreement_case`) | ¿Se cumplió esta obligación? | Lo abre la parte a la que se le debía. |
| **Conclusión sobre el efecto** (`FINDING`) | Qué pasó con la obligación y cómo se repara: `ESTABLISHED`, `PARTIALLY_ESTABLISHED` o `NOT_ESTABLISHED`; corrección económica `NONE` o `FULL_REVERSAL`; otra reparación en texto. No dice nada de intenciones. | El revisor que tomó el caso. |
| **Atribución de conducta** (`ATTRIBUTION`) | Una pregunta aparte: ¿la causó la conducta de alguien? `NO_FAULT`, `GOOD_FAITH_THIRD_PARTY`, `GOOD_FAITH_ERROR` o `BREACH`. Un tercero de buena fe es una respuesta válida. | Un revisor, sólo con la conclusión firme y si el efecto se estableció. Es opcional. |
| **Sanción** (`SANCTION`) | Sólo `WARNING`. No cambia ningún saldo, crédito, historia ni perfil. | Un revisor, sólo sobre una atribución **firme** de `BREACH`, y siempre de forma expresa. |
| **Apelación** (`agreement_case_appeal`) | ¿Se mantiene la decisión? `UPHELD` u `OVERTURNED`. | La presenta una parte dentro del plazo; la decide un revisor distinto del autor de la decisión. |

**Nada ocurre automáticamente.** Una decisión es firme sólo por un acto humano: una apelación confirmada, las dos partes aceptándola, o un revisor confirmándola cuando el plazo de apelación terminó sin apelación. El paso del tiempo por sí solo no cambia nada. Una atribución de incumplimiento no sanciona a nadie por sí misma, y un saldo nunca es una acusación.

Una conclusión firme no cambia el estado de la obligación, que sigue siendo función de las declaraciones de las partes (`AGREEMENT_FULFILMENT.md`). La conclusión es otro objeto y se muestra al lado.

## Quién revisa

La comunidad nombra a sus revisores (`stir.case_reviewer`), como el electorado de `ORDINARY_GOVERNANCE.md`: un registro propio de STIR, nunca derivado de roles de plataforma. Lo cambia la autoridad comunitaria (`ReferenceService.requireCommunityAuthority`): el propietario o administrador del tenant mientras la gobernanza ordinaria esté apagada, o un miembro del electorado si está encendida. **Un superusuario de plataforma no es autoridad comunitaria** y recibe 403.

- Al menos dos revisores: una apelación siempre la decide alguien distinto del autor.
- Cada revisor debe tener activado el intercambio económico: firma las correcciones con su propia clave.
- La configuración fija también el **plazo de apelación** (1 a 720 horas) y el **umbral de firmas** para una corrección.
- Nombrar revisores registra en osTRIS una versión nueva de la **autoridad de resolución** de la comunidad, con los controladores de esos revisores y ese umbral. La anterior deja de ser efectiva desde la siguiente secuencia.
- La lista de revisores y el umbral son públicos para la comunidad (`GET /case-review`), como pide R14: quién decide qué tiene que poder describirse.

Un revisor nunca revisa un caso en el que es parte: para ese caso es sólo parte.

## El camino de un caso

```
OPEN ──take──▶ UNDER_REVIEW ──finding──▶ FINDING_ISSUED ──appeal──▶ APPEALED
  │                 ▲                          │                       │
  │                 └──── OVERTURNED ◀─────────┼───────────────────────┤
  │                                            │ waive×2 / plazo + revisor / UPHELD
  └──withdraw──▶ WITHDRAWN                     ▼
                                           RESOLVED ──▶ atribución, sanción, corrección
```

- **Abrir:** sólo la parte a la que se le debía la obligación (`OBJECT`, `OWNER_TASKS`, `INITIATOR_TASKS`), y sólo si no está `DONE` ni `SUPERSEDED`. El intercambio económico no se reclama: osTRIS es su prueba. Hay un caso vivo como mucho por obligación. La otra parte recibe `CASE_OPENED`.
- **Pruebas:** cualquiera en el caso aporta pruebas, que nunca se editan. `REVIEWERS_ONLY` sirve para datos privados (una dirección, un documento): la otra parte ve que se aportó una prueba, pero no su contenido.
- **Tomar:** un revisor que no es parte.
- **Conclusión:** la emite el revisor que tomó el caso. Si ordena `FULL_REVERSAL`, STIR registra en osTRIS una base `FINAL_DISPUTE_RESOLUTION` en estado `PENDING`.
- **Apelar o aceptar:** cada parte, dentro del plazo. Si un revisor distinto del autor confirma la apelación, la decisión queda firme. Si la revoca, la base de osTRIS pasa a `OVERTURNED` y el caso vuelve a revisión con el revisor que decidió la apelación, nunca con el autor.
- **Firmeza:** al quedar firme la conclusión, la base de osTRIS pasa a `FINAL`, la contención se levanta y el caso queda `RESOLVED`. La atribución y la sanción, si las hay, siguen su propio camino de firmeza y apelación sobre un caso ya resuelto.
- **Retirar:** quien abrió el caso, mientras la conclusión no sea firme. Sirve también para un arreglo entre las partes.

## Contención

Un revisor puede pausar, mientras se revisa el caso, **la ejecución y las enmiendas del Agreement**: el `commit` y las firmas del intercambio económico, y proponer o aceptar enmiendas, responden `409 CASE_CONTAINMENT_ACTIVE`. Tiene motivo, autor y fecha de fin (30 días como mucho), se levanta a mano o al resolverse el caso, y nunca se borra. **Nunca afecta a una persona, una cuenta ni un saldo.**

## Reparar el efecto económico

Si la conclusión firme ordena `FULL_REVERSAL` y el intercambio está `COMMITTED`, un revisor prepara la corrección:

1. STIR propone en osTRIS una transacción `REVERSAL` con `reversesTransactionId` = la transacción original y la base `FINAL_DISPUTE_RESOLUTION` del caso. Las entradas son exactamente las originales negadas, en su orden. **Nunca un `EXCHANGE`.**
2. Cada revisor pide el texto exacto que firma (`GET .../signing-payload`, que osTRIS devuelve bajo el dominio `OSTRIS:TX:GOVAUTH:V1`), lo firma en su dispositivo con su propia clave y lo envía. STIR sólo transmite la firma. Una parte nunca firma la corrección de su propio acuerdo.
3. Con el umbral de firmas alcanzado, STIR pide el `commit`. osTRIS comprueba en la secuencia de commit la autoridad efectiva, la base `FINAL` y cada firma. Si falta algo, el intento queda registrado y la corrección sigue preparada.

El intercambio original sigue `COMMITTED`: la reversión es un segundo asiento enlazado, como exige el protocolo. El tablero muestra la corrección junto a la obligación económica (`corrections` en `GET /obligations`).

La versión 0.4 sólo hace **reversiones completas**. Una corrección parcial en osTRIS es `SETTLEMENT`, que `CLAUDE.md` mantiene como SPEC GAP para otro uso (la comisión de mercado); no se envía aquí. Un efecto establecido en parte puede repararse por otra vía descrita en la conclusión, o con un acuerdo nuevo.

## Contrato nuevo en osTRIS

osTRIS ya validaba un `REVERSAL` adjudicado, pero sólo los fixtures podían crear la autoridad y la base. La rama `claude/dispute-resolution-contract` añade la superficie mínima:

- `POST /api/ostris/communities/{c}/resolution-authorities` (`OSTRIS_COMMUNITY_MANAGE`) y `GET .../current`.
- `POST /api/ostris/communities/{c}/resolution-bases` y `POST .../{id}/finality` (permiso nuevo `OSTRIS_RESOLUTION_RECORD`).
- `GET /api/ostris/transactions/{id}/governance-payload`.
- `V8__resolution_lifecycle_guards.sql`: la autoridad no cambia salvo una revocación, sus controladores son fijos, y la base sólo va de `PENDING` a `FINAL` o `OVERTURNED` (y de `FINAL` a `OVERTURNED`); nada se borra.

Ninguno de estos endpoints autoriza una transacción. **La firmeza que STIR registra es una afirmación de quien la llama, no una prueba verificable** del proceso que la produjo. Por eso esto **no cierra el SPEC GAP 1** de `CLAUDE.md` (contrato verificable de resolución final): `REPLACE_CONTROLLER` sigue bloqueado.

## API

Bajo `/api/stir/tenants/{tenantId}`. Lectura con `stir.agreements.read`; escritura con `stir.agreements.fulfil`. Quién puede hacer cada cosa (parte, revisor, autoridad comunitaria) se decide en el servicio, así que un superusuario que no es ninguna de ellas recibe 404 o 403 como cualquiera.

- `GET /case-review`, `PUT /case-review` con `{reviewerUserIds, threshold, appealWindowHours}`.
- `GET /cases`: `{asParty, toReview}`.
- `GET /agreements/{id}/cases`, `POST /agreements/{id}/cases` con `{obligationId, claim, statement}`.
- `GET /cases/{id}`.
- `POST /cases/{id}/evidence`, `/take`, `/containment`, `/containment/lift`, `/findings`, `/attributions`, `/sanctions` y `/withdraw`.
- `POST /cases/{id}/decisions/{decisionId}/waive`, `/appeal` y `/finalize`.
- `POST /cases/{id}/appeals/{appealId}/decide`.
- `POST /cases/{id}/remedies`, `GET /cases/{id}/remedies/{remedyId}/signing-payload`, `POST .../signatures` y `POST .../commit`.

Los revisores, además, necesitan los permisos de osTRIS de las llamadas que STIR hace con su sesión: `OSTRIS_RESOLUTION_RECORD` y los de transacción. Quien nombra a los revisores necesita `OSTRIS_COMMUNITY_MANAGE`.

`GET /api/stir/instance` añade `agreementCasesVersion: 1`. Las notificaciones nuevas son `CASE_OPENED` y `CASE_UPDATED`.

## Base de datos

`V24__agreement_cases.sql` crea:

- `case_reviewer`;
- `case_review_settings`;
- `agreement_case`;
- `agreement_case_event`, la cronología, sólo inserción;
- `agreement_case_evidence`, sólo inserción;
- `agreement_case_decision`, con los tres tipos de decisión y restricciones por tipo;
- `agreement_case_appeal`;
- `agreement_case_containment`;
- `agreement_case_remedy`.

Los disparadores congelan lo que no debe cambiar:

- un caso conserva partes, obligación y relato, y un caso cerrado no se reabre;
- una decisión sólo cambia de estado, y `FINAL` y `OVERTURNED` son terminales;
- una apelación se decide una vez;
- una contención levantada no vuelve;
- una corrección conserva su intención en osTRIS, y una ejecutada no cambia;
- el historial de revisores no se borra.

Índices únicos parciales garantizan un caso vivo por obligación, una decisión vigente de cada tipo por caso, una apelación por decisión y una corrección por conclusión. Todo tiene RLS por tenant, y las nueve tablas están registradas como `COVERED` en el dominio `AGREEMENT_ECONOMIC` de la auditoría.

## Lo que no hace

- No hay puntuación ni reputación: un caso no aparece en ningún perfil ni en la historia de cumplimiento.
- No hay sanciones automáticas, ni otra sanción que la advertencia.
- No hace correcciones económicas parciales (ver arriba) ni `PENALTY` o `RESTITUTION` en osTRIS.
- No revisa quién decide el suelo de crédito: eso es la 0.5.
- No corrige tramos externos: lo hace cada distribución en su sistema.
