# STIR Network — arquitectura comunitaria experimental

STIR Network es la propuesta de infraestructura XRPL para evidencias de la comunidad STIR. La separación de producto prevista es **STIR → osTRIS → IDAX Ledger → XRPL**: STIR y sus usuarios no administran directamente validators ni listas de confianza.

La versión **v0.1 es un diseño experimental abierto a revisión pública**, no un anuncio de lanzamiento. Propone cinco validators privados, dos hubs públicos sin voto y un nodo histórico. La etapa inicial estaría operada por el proyecto y **no se presenta como descentralizada**. El objetivo es evolucionar hacia nodos y validators gestionados por personas y organizaciones independientes. Un operador externo puede comenzar con un nodo y pasar por **observation → candidate → trusted** mediante revisión explícita; conectarse a la red no concede confianza.

La decisión de crear una nueva génesis/`NetworkID` frente a continuar la red privada de laboratorio permanece **OPEN**, al igual que placement, recursos, custodia del publisher, manifiesto de bootstrap, RPO/RTO y plan de lanzamiento. Ninguno de estos documentos publicados contiene claves o secretos. Las ubicaciones, identidades operativas y procedimientos privados se mantienen fuera de esta carpeta.

## Leer el diseño

- [Arquitectura y topología](STIR_NETWORK_ARCHITECTURE_V0.1.md)
- [Modelo de confianza de validators](VALIDATOR_TRUST_MODEL_V0.1.md)
- [Ciclo de nodos y operadores](NODE_OPERATOR_LIFECYCLE_V0.1.md)
- [Principios de recuperación](DISASTER_RECOVERY_V0.1.md)
- [Modelo conceptual de amenazas](THREAT_MODEL_V0.1.md)
- [Decisiones arquitectónicas propuestas](NETWORK_ADRS_V0.1.md)
