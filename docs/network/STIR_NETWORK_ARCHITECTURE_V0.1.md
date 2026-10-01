# STIR Network Architecture v0.1 — diseño para revisión

**Estado:** READY para revisión arquitectónica; **no** autorización de despliegue. Fecha: 2026-10-01. Alcance: red XRPL experimental de la comunidad asociada a STIR. Esta propuesta no cambia servicios, DNS, claves ni datos.

## 1. Base verificada y límite de la propuesta

La evidencia de laboratorio aportada por el proyecto: XRPL source-built desde upstream tag `3.3.0`, commit `00a178fb92ca49521b937ae1a99d863765ea8a90`; Phase 3B y Phase 4 readiness PASS, 10/10 arranques en frío, bootstrap determinista, soak de 8 h, recuperación histórica, `MATCH`/`VALIDATED_MATCH` y rollback entre imágenes. La génesis, las claves y el `NetworkID` de la red privada de laboratorio **no quedan automáticamente designados** para una red comunitaria. Los resultados citados prueban el entorno ensayado, no rendimiento o disponibilidad en los emplazamientos futuros.

Contrato de producto: `STIR → osTRIS → IDAX Ledger → XRPL`. STIR no conoce peers, validators ni UNL. osTRIS no gestiona confianza XRPL. IDAX Ledger es la capa prevista para seleccionar endpoints, observar salud, comparar `(NetworkID, ledger index, validated hash)` entre nodos y suspender una lectura o envío ante discrepancias; mantiene la semántica de Proof y no convierte `SUBMITTED` en `VALIDATED`. La lectura de Proof histórico requiere un nodo con el ledger correspondiente, no sólo un nodo sincronizado hoy. Son **capacidades objetivo**, no afirmaciones sobre el código actual.

## 2. Topología inicial recomendada

**Cinco validators privados propuestos, en hosts separados bajo administración inicial del proyecto**, más dos hubs públicos separados y un nodo histórico sin voto. Es una demo experimental con tolerancia a un validator caído en el caso nominal; **sigue siendo una red de un operador**. El placement definitivo queda abierto hasta medir recursos y dependencias. Si los hosts disponibles no soportan los roles junto a sus otras cargas, se reduce el alcance de la demo o se aplaza, sin etiquetar un hub como validator.

| Rol lógico | Placement público propuesto | Exposición | Riesgo a medir |
|---|---|---|---|
| V-1…V-5 | hosts separados; dominios de infraestructura por decidir | peering privado; sin API pública | energía, red, proveedor y control humano compartidos |
| H-1 y H-2 | dos emplazamientos por decidir | peer público; API limitada si se ofrece | rutas de acceso correlacionadas |
| archive | emplazamiento y capacidad por decidir | consulta histórica controlada; sin voto | pérdida de historia |
| sitio STIR | independiente del consenso | HTTPS público | dominio o hosting del sitio |

```text
       Usuarios → STIR → osTRIS → IDAX Ledger ──┬→ H-1 (lectura/envío)
                                                 └→ H-2 (lectura/envío)
                      monitor privado ────────────→ V-1…V-5 (placement abierto)
   peers externos ←→ H-1 ←→ H-2 ←→ hubs futuros de terceros
                        ↖  peering controlado con validators  ↗
   snapshots/archivo ← nodo histórico independiente, sin voto
   lista firmada ← publisher de confianza; réplicas web/Tor/IP son transporte
```

Varios servidores de una misma instalación no son dominios de fallo independientes. Varias VMs de un mismo proveedor tampoco representan varios proveedores. Energía, red, cuenta de infraestructura, operador humano, DNS y dominio pueden seguir correlacionados. El protocolo peer necesita rutas y controles de red propios, sujetos a prueba. Un fallo del operador o de sus credenciales puede afectar a los cinco votos. Ningún dominio de infraestructura debe convertirse en la única ruta de peering para todos los validators.

### 3, 4 o 5 validators

La tabla muestra el umbral nominal `ceil(0,8 × N)` para un conjunto confiado común y sin ajuste por Negative UNL. XRPL puede recalcular el quorum con Negative UNL; verificar el valor **real** mediante `server_info`/`validators` y probar particiones antes de cualquier lanzamiento. [Consenso](https://xrpl.org/docs/concepts/consensus-protocol), [Negative UNL](https://xrpl.org/docs/concepts/consensus-protocol/negative-unl).

| N | Umbral nominal | Caídas simultáneas toleradas nominalmente | Coste y operación | Valor aquí |
|---:|---:|---:|---|---|
| 3 | 3 | 0 | mínimo; mantenimiento coordinado congela progreso | válido para laboratorio, frágil para demo pública |
| 4 | 4 | 0 | más coste sin ganancia nominal de disponibilidad | no recomendado |
| 5 | 4 | 1 | cinco custodias, upgrades y monitorización | recomendado para demo, no aumenta independencia humana |

Quorum de firmas no equivale a tolerancia a una caída de proveedor, instalación o del operador. Si una dependencia compartida elimina dos o más votos, los restantes no alcanzan el umbral nominal de 4/5. Negative UNL no es un plan de alta disponibilidad ni una garantía de recuperación de un fallo correlacionado. Mantener la UNL idéntica o con solapamiento probado entre participantes es requisito de seguridad; listas divergentes pueden producir visiones divergentes. [Solapamiento](https://xrpl.org/docs/concepts/consensus-protocol/consensus-protections).

## 3. Hubs, validators, nombres y acceso

Los hubs admiten peers descubiertos y absorben conexiones públicas. Los validators usan `peer_private=1`, peers fijos de al menos dos rutas/emplazamientos, firewall de entrada sólo para peers conocidos y RPC/admin sólo en red administrativa o loopback. No se publica la IP del validator ni se promete ocultarla frente a un hub deshonesto. No se sirve API pública ni explorer desde un validator. [Guía oficial de private server](https://xrpl.org/docs/infrastructure/configuration/peering/configure-a-private-server), [guía de validator](https://xrpl.org/docs/infrastructure/configuration/server-modes/run-xrpld-as-a-validator).

Convención propuesta, **sin nombres registrados**: `nNNNN.<dominio-comunitario>` para hubs públicos; nombres distintos para API/explorer, distribución de lista y estado público. Un validator tiene un identificador lógico estable `validator:<master-public-key>` y un alias descriptivo, jamás depende de un hostname como identidad. Hostnames y direcciones privadas de validators pertenecen al inventario administrativo no público. Identidad de operador: registro verificable separado, potencialmente con varios dominios/contactos.

## 4. Descubrimiento, recursos y observabilidad

El manifiesto **de bootstrap comunitario**, distinto de la lista XRPL, será un artefacto público firmado, serializado canónicamente y versionado: `schemaVersion`, `networkName`, `NetworkID`, hash de génesis/checkpoint verificable, secuencia y expiración, hubs con endpoint peer/API y roles, claves públicas del publisher, referencia/digest **y claves maestras públicas del conjunto trusted vigente**, versión compatible de software, fuente/tag/commit, digest de imagen y rutas de recuperación/contacto. Sólo claves públicas y datos de red; nunca tokens, seeds, IP de administración o rutas de backup. Firma con una clave de publicación separada de validator/master/anchor; clientes fijan su huella por canal independiente. Un manifiesto viejo o de otra red falla cerrado. La lista firmada XRPL sigue siendo la autoridad de UNL que cada servidor configura, no este documento de descubrimiento; el conjunto del manifiesto se contrasta con esa lista.

Perfiles **experimentales pendientes de benchmark**: demo node (peer/RPC limitado), hub público (más CPU, ancho de banda y límites API), validator dedicado (prioridad a latencia/estabilidad, sin tráfico público), archive (almacenamiento e IOPS dominantes, sin voto). Los ~175–227 MiB observados en validators de tests son una muestra de esa carga; no son un mínimo. La [guía oficial](https://xrpl.org/docs/infrastructure/installation/system-requirements) indica requisitos mucho mayores para Mainnet, que tampoco deben copiarse sin medición a esta red. En una matriz de carga por rol medir p50/p95/p99 de RAM/CPU, pico de arranque, GB/día de disco, IOPS sostenidas y latencia, bytes/s de entrada/salida, tiempo de arranque y recuperación, desfase NTP, peers y `complete_ledgers`. Ensayar carga de Proof, consultas históricas, interrupción y restore en cada tipo de host durante varios días; publicar después un **STIR XRPL Experimental Minimum Profile** con margen y versión de software. Si archive resulta inviable, retener al menos un snapshot histórico verificado en dos emplazamientos antes de prometer verificación histórica continua.

Monitorización privada: `server_info` (`server_state`, `validated_ledger`, `complete_ledgers`, `validation_quorum`, expiración de lista), `validators`/`validator_list_sites`, peers, propuesta/validaciones propias, latencia y NTP, uptime/restart, disco/IOPS, comparación de hash validado por altura entre hubs, y Proofs históricos canario con `VALIDATED_MATCH`. Alertas por validator offline/no proposing, pérdida de peers, huecos, divergencia, fallo histórico, expiración de lista, presión de disco/cambio de identidad. Panel público sólo publica salud agregada, altura/hash ya públicos y avisos; no credenciales, IP internas, rutas admin ni logs sensibles. Una divergencia se registra y se escala; IDAX Ledger no debe ocultar el nodo defectuoso tras un simple failover.

## 5. Evolución y decisiones abiertas

Fases con puertas verificables: **A** proyecto opera cinco validators privados y DR ensayado; **B** hubs/nodos públicos de terceros sin voto y perfil mínimo medido; **C** candidates externos observados sin alta automática; **D** al menos cuatro validators confiados de operadores realmente independientes y proveedores/jurisdicciones diversas, con pruebas de incidentes y quorum bajo fallos; **E** política de confianza y publisher bajo gobernanza distribuida con procedimientos auditables. Nunca describir A–C como descentralización real. Un operador puede participar sin ser autoridad constitucional o administrador de STIR.

Decisiones **OPEN** previas al despliegue: (1) nueva génesis/`NetworkID` e identidades frente a continuidad de la red privada; se prefiere estudiar una red nueva que preserve la anterior como laboratorio e historia verificable, sin decisión formal; (2) placement final de validators, hubs y archivo; (3) benchmark y perfil mínimo de recursos; (4) custodia/recuperación del publisher; (5) formato y claves del manifiesto de bootstrap; (6) retención y RPO/RTO; (7) ubicación independiente de backups; (8) límites de API/peering y plan de lanzamiento/rollback. Véanse [modelo de confianza](VALIDATOR_TRUST_MODEL_V0.1.md), [ciclo de operadores](NODE_OPERATOR_LIFECYCLE_V0.1.md), [DR](DISASTER_RECOVERY_V0.1.md), [amenazas](THREAT_MODEL_V0.1.md) y [ADRs](NETWORK_ADRS_V0.1.md).
