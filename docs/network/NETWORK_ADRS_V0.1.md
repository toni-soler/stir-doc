# STIR XRPL network — ADRs v0.1

Todos los ADRs están **propuestos para revisión**; ninguno autoriza despliegue. Véanse [arquitectura](STIR_NETWORK_ARCHITECTURE_V0.1.md), [confianza](VALIDATOR_TRUST_MODEL_V0.1.md), [lifecycle](NODE_OPERATOR_LIFECYCLE_V0.1.md), [DR](DISASTER_RECOVERY_V0.1.md) y [amenazas](THREAT_MODEL_V0.1.md).

## ADR-01: cinco validators iniciales

**Context:** con quorum nominal del 80 %, 3/3 y 4/4 no toleran una caída; 5/4 tolera una. **Decision:** cinco validators privados si el benchmark de los hosts lo permite; llamar a la fase demo de un solo operador. **Alternatives:** tres para laboratorio; cuatro sin beneficio nominal; posponer la demo si faltan recursos. **Consequences:** más coste, mantenimiento y custodia; proveedor/operador pueden seguir compartidos; verificar quorum real y Negative UNL. **Status:** propuesto.

## ADR-02: hubs públicos y validators privados

**Context:** API/peer públicos amplifican carga y DDoS sobre consenso. **Decision:** hubs sin voto expuestos; validators con `peer_private`, peers fijos, RPC/admin restringidos y al menos dos rutas. **Alternatives:** validators públicos; un solo hub; proxy en cada host. **Consequences:** más VMs y configuración; los hubs ven IP de validators y pueden cortar peering, por lo que se diversifican. **Status:** propuesto.

## ADR-03: identidad del validator separada

**Context:** host, operador y clave de validación cambian a ritmos distintos. **Decision:** registro de operador, master key offline, token/clave activa y hostname independientes. **Alternatives:** seed en servidor o identidad igual a DNS. **Consequences:** ceremonia y backups de master, rotación y revocación documentadas; migración de host sin cambio de master. **Status:** propuesto.

## ADR-04: admisión explícita a trusted

**Context:** conectar un nodo o emitir validaciones no lo hace confiable. **Decision:** node → observation → candidate → revisión → trusted, con criterios medibles e independencia evaluada. **Alternatives:** alta automática, invitación discrecional sin evidencia. **Consequences:** onboarding más lento y decisiones auditables; riesgo de concentración visible. **Status:** propuesto.

## ADR-05: publisher evolutivo

**Context:** una lista común mejora solapamiento, pero una sola custodia centraliza cambios. **Decision:** Phase A publisher del proyecto con lista firmada/replicada/expirable; Phase B proceso de confianza distribuido y publishers ensayados; Phase C operadores independientes eligen. **Alternatives:** UNL estática para siempre; múltiples publishers sin política de umbral. **Consequences:** hay que probar expiración, pérdida y transición de claves; mirrors no sustituyen gobernanza. **Status:** propuesto.

## ADR-06: backups por clases

**Context:** un backup de DB no restaura Proof histórico ni credenciales. **Decision:** datos, credenciales, configuración/políticas y fuente/procedencia separados; cifrado y copias en dominios de fallo independientes; restauración periódica. **Alternatives:** snapshots de VM únicos; regenerar identidades. **Consequences:** coste de archivo y custodia; verificación de hashes tras restore. **Status:** propuesto.

## ADR-07: independencia de dominio

**Context:** `stir.es` y DNS pueden perderse. **Decision:** claves públicas fijadas fuera de DNS, manifiesto firmado y versionado, mirrors en dominio/IP/Git y contactos fuera de banda. **Alternatives:** DNS como root of trust; URL única. **Consequences:** distribución inicial de huellas y procedimientos de transición; hostname nunca identifica validator. **Status:** propuesto.

## ADR-08: Tor como transporte opcional

**Context:** bloqueo de dominio y privacidad pueden justificar rutas alternativas; latencia puede dañar consenso. **Decision:** probar onion para frontend/API y distribución de artefactos firmados; peer protocol sólo laboratorio medido. **Alternatives:** Tor obligatorio o ausencia total. **Consequences:** operación adicional sin dependencia crítica. **Status:** propuesto.

## ADR-09: evolución de topología

**Context:** instalaciones y proveedores pueden concentrar fallos, y un operador controla inicialmente todo. **Decision:** introducir primero hubs externos, luego candidates y después validators confiados de varios operadores/proveedores; medir concentración antes de afirmar descentralización. **Alternatives:** añadir más VMs del mismo dueño; declarar descentralización por número de validators. **Consequences:** coordinación y confianza social más difíciles; una caída correlacionada puede detener el quorum en fase A. **Status:** propuesto.

## ADR-10: continuidad de génesis pendiente

**Context:** existe una red privada de laboratorio con historia de Proofs, pero no hay decisión de convertirla en red comunitaria. **Decision:** mantener **OPEN** la elección entre nueva génesis/`NetworkID` y continuidad; se prefiere examinar una red nueva preservando la anterior como laboratorio verificable, sin aprobar aún esa opción. **Alternatives:** heredar historia/identidades actuales; red nueva con archivo verificable de la antigua. **Consequences:** el manifiesto público queda conceptual; una red nueva no puede reclamar las Proofs de la anterior como propia. **Status:** abierto.
