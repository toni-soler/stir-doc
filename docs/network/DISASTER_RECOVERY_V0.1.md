# Disaster recovery v0.1 — continuidad de la comunidad

**Estado:** diseño, sin backups nuevos ni ejercicios ejecutados. `stir.es` es un punto de acceso, no la identidad de la red. La identidad reproducible exige el mismo génesis/`NetworkID`, historia verificada y políticas/listas firmadas compatibles. Recuperar una base de datos sin los ledgers que contienen sus Proofs no conserva la verificabilidad histórica.

## Cuatro clases de backup

| Clase | Contenido | Custodia |
|---|---|---|
| Datos | PostgreSQL IDAX Ledger y, según despliegue real, osTRIS/STIR; object storage/versiones; NuDB/SQLite/history o snapshots XRPL consistentes | copias cifradas, checksums y emplazamientos en dominios de fallo independientes |
| Credenciales | validator master offline, tokens activos por separado, anclaje/anchor credentials, claves de publisher y de manifiesto, secretos DB | cifrado separado, acceso dual para master, inventario de rotación; nunca en imagen/repo ni en backup ordinario sin cifrar |
| Fuente/procedencia | manifiesto de upstream/tag/commit, imagen/digest y build provenance, locks/dependencias, scripts de build y hashes | copias firmadas, almacenamiento independiente |
| Configuración/política | génesis, `NetworkID`, peer/UNL sin secretos, configuración de retención, manifiesto de bootstrap, listas y políticas firmadas, playbooks | versionada y firmada; secretos inyectados sólo al restaurar |

Capturar estado de PostgreSQL y object storage con punto temporal conocido; registrar high-water marks de Proof/ledger para reconciliación. Hacer snapshots XRPL tras parada o con mecanismo de consistencia probado, nunca asumir que copiar NuDB/SQLite en caliente es válido. Dos copias de historia/snapshot en dominios físicos distintos y canarios de ledgers antiguos. La reconstrucción desde peers sólo sirve si conservan exactamente la historia requerida; un stock node con `online_delete` puede no conservarla. [Full history oficial](https://xrpl.org/docs/infrastructure/configuration/data-retention/configure-full-history).

## DR bundle conceptual y prueba

Bundle cifrado/versionado: índice firmado; `source-manifests/`; `build-provenance/`; `config-public/`; `signed-policies-and-lists/`; `postgres/`; `object-storage/`; `xrpl-history/`; `credential-vault/` cifrado por separado; `restore-scripts/`; `verification-scripts/`; checksums, fechas, custodios, RPO/RTO y hashes de Proof canario. El índice público sólo muestra hashes y referencias seguras, nunca ubicación de secretos. Verificar compatibilidad de binarios y NetworkID antes de arrancar.

Prueba trimestral propuesta en infraestructura aislada: restaurar datos, historia y configuración; recuperar al menos un ledger histórico con mismo hash; restaurar identidades sólo cuando se autorice y evitando dos validators simultáneos con la misma clave; alcanzar quorum de prueba; comprobar tres o más Proofs preexistentes como `MATCH`/`VALIDATED_MATCH` con mismo tx hash, ledger index/hash; probar lectura desde dos hubs; registrar tiempo/RPO/RTO y fallos. Ensayar semestralmente pérdida de dominio y publisher, con nueva URL pero mismas claves públicas ancladas fuera de DNS. No tocar la red viva en esos ejercicios.

| Pérdida | Recuperación prevista | Límite |
|---|---|---|
| `stir.es` / DNS / web | dominios alternativos, IP bootstrap, mirrors firmados y contactos fuera de banda | usuarios deben conocer la huella de firma por canal independiente |
| proveedor de infraestructura | nodos y copias en otros proveedores; recuperar datos STIR/osTRIS si allí residían | la pérdida de dos o más votos puede detener el quorum nominal |
| instalación / conectividad / energía | copias fuera de la instalación; reconstruir hosts con identidad segura | varios votos correlacionados pueden detener el quorum nominal |
| uno o varios validators | token nuevo/host nuevo si master sana; revocar y reemplazar si master comprometida | cambio de UNL requiere coordinación y solapamiento |
| publisher | servir lista firmada replicada hasta expiración; recuperar clave offline o transición explícita a nuevo publisher | sin custodia o transición antes de expiración la red puede dejar de validar |
| pérdida de historia | restaurar snapshots/archivo y verificar hashes y Proofs | si ninguna copia conserva ledgers antiguos, no prometer historia intacta |

IP directa, dominios alternativos, repositorios Git y Tor son canales de descubrimiento, no raíces de confianza. El manifiesto y lista se verifican contra claves públicas conocidas fuera de esos canales. Tras un desastre no se modifica el pasado del ledger para aparentar continuidad; si la génesis cambia, se reconoce una red nueva y se preserva acceso verificable a la anterior.
