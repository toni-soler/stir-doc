# Threat model v0.1

**Activos:** historia y hashes validados, conjuntos confiados/clave publisher, claves master/token/anchor, bases de Proof, objetos, configuración, fuente/procedencia y disponibilidad. **Adversarios:** terceros de Internet, candidato malicioso, proveedor/administrador comprometido y coordinación de insiders. Fase A es una demo de un operador, por lo que varios validators no impiden captura de ese operador.

| Amenaza | Efecto plausible | Defensa proporcionada para demo / señal |
|---|---|---|
| DDoS a hub/API | pérdida de acceso o peering | hubs separados, límites API, validators privados, varias rutas; alertar por peers/latencia |
| Compromiso de validator/host | voto erróneo, robo de token, exposición de peers | token separado de master offline, firewall, reconstrucción, rotación y revisión de validaciones |
| Compromiso de operador | control correlacionado de varios votos | declarar centralización, separación futura de custodios; ninguna VM adicional corrige esto |
| DNS takeover, secuestro de dominio o bloqueo gubernamental | dirigir usuarios a endpoints falsos o cortar acceso | firmas fijadas fuera de DNS, dominios/IP/Tor alternativos, mirrors; no confiar en HTTPS/DNS como autoridad de UNL |
| Cuenta de proveedor comprometida | pérdida/copia de varias VMs y servicios | backups fuera del proveedor, MFA, permisos mínimos, alertas; topología futura multi proveedor |
| Candidato malicioso o conjunto coordinado | intento de influir en consenso al entrar en UNL | observación sin voto, revisión de independencia, límites de concentración y retirada; medir solapamiento |
| Publisher comprometido | lista falsa, exclusión/inclusión de claves | firma/custodia offline, secuencia/expiry, control de cambio, mirrors, aviso y transición explícita |
| Robo de backup | claves/datos de usuarios expuestos | cifrado separado de credenciales, acceso dual, restore auditado, prueba de lectura |
| Pérdida de credenciales | imposibilidad de rotar o demostrar identidad | copias offline geográficamente separadas y restore probado; revocación/reemplazo si se pierde master |
| Supply chain/update malicioso | validaciones divergentes o extracción de claves | pin upstream/commit/digest, procedencia, staging y rollback probado; comparar hashes entre versiones |
| Reloj/partición/red | votos tardíos, pausa o vista divergente | NTP, peers en rutas distintas, comparación de hashes, no enviar Proof sobre divergencia |

Tor: **A** frontend/API/explorer onion como acceso alternativo opt-in, si el riesgo de bloqueo lo justifica; **B** mirror onion de manifiesto/lista *firmados* para recuperación; **C** peer protocol sobre Tor sólo experimento aislado, midiendo latencia, estabilidad y pérdida de peers. Tor no es ruta de consenso obligatoria ni reemplaza firma o múltiples operadores. Un servicio onion nuevo no recibe confianza por tener dirección onion. La fase demo prioriza firewall, claves offline, backups y monitorización; no presupone HSM, anti-DDoS global o anonimato perfecto.
