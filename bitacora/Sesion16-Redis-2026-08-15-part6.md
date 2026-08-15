# Sesión Redis — 2026-08-15 (part 6)

**Bloque 4 (opcional del arco Redis) — Parte 2 de N: Replicación master-réplica.**

Duración efectiva: ~2h, sábado mañana. Energía media-alta (buen descanso previo), ritmo sostenido.

Contexto de sesión: continuación directa del bloque 4 tras la parte 1 (RDB/AOF de ayer). Scope acotado desde el arranque: solo replicación master-réplica esta sesión. Cache-aside queda para siguiente sesión (requiere código Java nuevo, tema propio). Decisión correcta a posteriori — la teoría + lab + gotchas de compose profiles + commit consumieron las 2h.

## Repo state al cerrar

- Rama `main`, HEAD en `db4ff84`, un commit por delante del `524f700` con el que arrancó la sesión. Sincronizado con `origin/main`.
- Working tree clean.
- Contenedores `document-analyzer-redis` y `document-analyzer-redis-replica` levantados con profile `ha-lab` activo al cerrar. Recomendación: `docker compose --profile ha-lab down` al cerrar el portátil para liberar recursos.
- Directorio `redis-data/` intacto con ficheros heredados de la sesión anterior + escritura de la sesión de hoy en el RDB actualizado.

## Commits generados en la sesión

1. `db4ff84 feat(ha-lab): add optional Redis replica behind ha-lab profile` — añade servicio `redis-replica` al `docker-compose.yml` configurado con `--replicaof redis 6379`, expuesto en puerto host 6380, sin volumen (rehidrata desde master), gated tras el profile `ha-lab`. Mensaje bilingüe extenso documentando validaciones empíricas: propagación sub-milisegundo, READONLY en escrituras a réplica, comportamiento tras caída de master, PSYNC2 preservando replid/offset tras reinicio.

## Conceptos cubiertos

### Replicación master-réplica — modelo mental base

- **Qué**: dos o más instancias Redis simultáneas, una designada master (única aceptando escrituras), el resto réplicas manteniendo copia continua del dataset del master.
- **Vocabulario**: `master` / `replica` (antes `slave`, renombrado en 2018). `INFO replication` mantiene los nombres antiguos por retrocompatibilidad con herramientas de monitorización; cohabitan `slave_repl_offset` y `replica_announced` en la misma salida.
- **Tres motivos independientes para montarla**:
  1. **Read-scaling**: repartir lecturas entre master y réplicas, dejando escrituras solo al master. Útil en cargas 90% read / 10% write típicas de caché.
  2. **Alta disponibilidad (HA)**: si el master muere, una réplica puede ser promovida (manual o automáticamente vía Sentinel/Cluster). Sin réplica, master caído = downtime.
  3. **Backups sin impactar producción**: `BGSAVE` en réplica no castiga al master.
- Los tres son independientes: cualquiera puede motivar la arquitectura por sí solo. En banca típicamente coexisten read-scaling + HA.

### Replicación asíncrona — implicación crítica

- El master responde `OK` al cliente **antes** de que la escritura llegue a la réplica. Propagación en background por el socket TCP.
- Ventana típica en red sana: milisegundos (sub-milisegundo en red Docker local, decenas-centenas ms en réplicas geográficamente distribuidas).
- **Consecuencia**: si el master muere justo después de confirmar una escritura pero antes de propagarla, y se promociona la réplica, esa escritura se pierde.
- Redis prioriza latencia sobre garantía de replicación. Postgres, por comparación, ofrece modos síncronos (master espera confirmación de réplica antes de responder). Redis no.
- **Regla de decisión de arquitectura**: si perder un dato requiere auditoría (transferencias, movimientos contables), NO vive en Redis. Si perderlo solo significa recalcularlo (saldos derivados de la BD relacional, sesiones, caché), Redis es candidato. Redis nunca es fuente de verdad de nada crítico.

### Sincronización inicial — full sync vía RDB por TCP

Cuando una réplica arranca contra un master con dataset ya cargado:

1. Réplica se conecta al master y solicita sincronización.
2. Master lanza `BGSAVE`: fork, proceso hijo escribe snapshot RDB (mismo mecanismo de persistencia local, no bloquea al master padre).
3. Durante el BGSAVE, master acumula las escrituras nuevas en un buffer dedicado a esa réplica (**replication backlog**).
4. RDB listo → master lo transmite por la conexión TCP a la réplica.
5. Réplica recibe el fichero, **borra su dataset previo** y lo carga en memoria.
6. Master envía a continuación el contenido acumulado en el buffer durante los pasos 2-5.
7. A partir de aquí, flujo continuo (stream de comandos, siguiente sección).

**Insight interview-ready**: el snapshot RDB en Redis cumple **doble función** — persistencia local en disco Y transferencia inicial full-sync a réplicas nuevas. Mismo mecanismo (BGSAVE con fork para no bloquear), dos canales distintos (filesystem vs TCP).

### Sincronización parcial — PSYNC

Optimización para desconexiones breves. La réplica ya sincronizada pierde momentáneamente la conexión (red flakea 5s). Redis evita el full sync completo si puede.

- Master mantiene un **replication backlog circular** (buffer de los últimos N MB de comandos propagados, default `repl_backlog_size:1048576` = 1 MB).
- Cada réplica lleva su **offset** actual (posición en bytes en el stream).
- Al reconectar, réplica envía `(replid, offset)` al master. Si el `replid` coincide (mismo stream, mismo master) y el offset está dentro del backlog, master envía solo el delta pendiente → **PSYNC**.
- Si el backlog ha rotado más allá del offset de la réplica, o el replid no coincide → **cae a full sync completo**.
- En producción con carga alta, subir `repl_backlog_size` a 256 MB o 1 GB para tolerar desconexiones más largas sin caer a full sync.

### PSYNC2 (Redis 4.0+) — preservación de identidad tras reinicio limpio del master

Descubierto empíricamente en el lab, contradijo mi predicción inicial. Mecanismo real:

- Cuando el master recibe SIGTERM (shutdown limpio), persiste `replid` y `master_repl_offset` **dentro del RDB** que escribe antes de morir.
- Al reiniciar, master genera un `replid` nuevo por seguridad (visible como `master_replid`), **pero preserva el antiguo como `master_replid2`** y el offset donde se quedó como `second_repl_offset`.
- Cuando la réplica reconecta y anuncia su último `(replid, offset)` conocido, el master mira su `master_replid2` y reconoce el stream. Hace **PSYNC en lugar de full sync**, aunque el master haya reiniciado.
- Verificado en output: `master_replid2:6a5f84a7…` = el replid del master antes del reinicio, `master_replid:8e9595e17e…` = generado en el arranque nuevo. La réplica no se resincronizó desde cero.

Es una optimización crítica: en producción con datasets grandes, un full sync tras reinicio controlado del master sería inaceptable operacionalmente.

### `INFO replication` — lectura de campos clave

**En el master**:
- `role:master`
- `connected_slaves:N` — número de réplicas conectadas activamente.
- `slave0:ip=X,port=Y,state=online,offset=Z,lag=W` — ficha de cada réplica. `state=online` en régimen; `wait_bgsave` o `send_bulk` durante full sync inicial. `lag=0` es réplica al día.
- `master_replid` — identificador único del stream de replicación, generado al arrancar.
- `master_repl_offset` — posición actual del stream en bytes, creciente con cada comando propagado.
- `master_replid2` + `second_repl_offset` — identidad del stream anterior tras reinicio limpio (PSYNC2). Vacíos (`0…0` y `-1`) si no ha habido reinicio.
- `repl_backlog_size` — tamaño del buffer circular para PSYNC. 1 MB default.

**En la réplica**:
- `role:slave`
- `master_host` + `master_port` — a quién considera su master (lo que se le pasó en `--replicaof`).
- `master_link_status` — `up` conectada, `down` desconectada.
- `master_last_io_seconds_ago` — segundos desde el último byte recibido del master. Valor `-1` = conexión caída.
- `master_link_down_since_seconds` — aparece solo cuando la conexión está caída. Contador que consumen las herramientas de monitorización (Prometheus) para disparar alertas.
- `master_sync_in_progress` — 1 durante full sync inicial, 0 en régimen.
- `slave_read_only:1` — modo read-only por defecto, rechaza escrituras con `READONLY`.

### Read-only en réplicas

- Default: `replica-read-only yes`. Intento de escritura en réplica → `(error) READONLY You can't write against a read only replica.`
- Existe flag `replica-read-only no` para casos muy específicos (scratchpad temporal en la propia réplica). En 99% de setups, incluyendo banca, no se toca.
- Propósito: prevenir divergencia. Sin este safeguard, una escritura local en la réplica crearía dos datasets divergentes que nunca reconcilian (la escritura local no sube al master, y cuando el master propague sus cambios, la réplica tendrá una mezcla incoherente).

### Sin failover automático en Redis puro

- Verificado empíricamente: `docker compose stop redis`, la réplica sigue viva.
- Campos que cambiaron en `INFO replication` de la réplica: `master_link_status:down`, `master_last_io_seconds_ago:-1`, apareció `master_link_down_since_seconds:23`.
- Campos que NO cambiaron: `role:slave`, `master_failover_state:no-failover`.
- **La réplica no se auto-promueve**. Sigue en modo `slave` read-only esperando a que el master vuelva. Lecturas sí, escrituras no.
- Estado híbrido "servicio degradado, no caído": la aplicación puede seguir leyendo sobre el último estado conocido, pero no escribir.
- Para failover automático hace falta capa por encima: **Redis Sentinel** (monitoriza topología master-réplica, hace failover automático) o **Redis Cluster** (Sentinel + sharding automático en 16384 hash slots).
- No entramos a montarlos en lab (cada uno es tema propio de 2-3 sesiones). Nivel conceptual suficiente para entrevista: saber que existen, qué problema resuelve cada uno, cuándo elegir uno u otro.

### Sharding (introducido conceptualmente, no en lab)

- **Sharding = partir horizontalmente el dataset entre N máquinas, cada una con un trozo distinto**.
- Contraste con replicación: replicación = N copias del mismo dataset (HA + read-scaling). Sharding = N particiones de un dataset dividido (escala de capacidad: RAM, throughput de escritura, tamaño del keyspace).
- Redis Cluster combina ambas: divide keyspace en 16384 hash slots entre N masters, cada master con sus propias réplicas.

## Laboratorio empírico (ejecutado)

**Fase 1 — añadir servicio `redis-replica` al compose**:
- Nombres: servicio `redis-replica`, container `document-analyzer-redis-replica`. Motivación real de la unicidad: identificador único a nivel del daemon Docker, no relacionado con HA.
- Puerto: `"6380:6379"` (host 6380 → contenedor 6379). Interno del contenedor sigue siendo 6379 (mapeo desacopla puerto real del puerto accesible desde host).
- **Sin volumen**: réplica rehidrata desde master vía full sync en cada arranque. Refuerza pedagógicamente que réplica = estado derivado, master = fuente de verdad.
- `command: redis-server --replicaof redis 6379`. Forma shell (string único). Con forma lista habría que separar `["redis-server", "--replicaof", "redis", "6379"]` como 4 elementos porque cada uno es un `argv` separado.
- `depends_on: redis: condition: service_started` para orden de arranque.

**Fase 2 — verificar sincronización inicial**:
- `docker compose up -d` → ambos contenedores UP.
- `KEYS *` en réplica devolvió las 5 claves heredadas del master (`aof:test:1/2/3`, `politica:test`, `test:persistencia`). Sin volumen en la réplica, esas claves solo pudieron llegar por full sync vía TCP desde el master. Verificación empírica del mecanismo.
- `SET test:manual "hola"` en réplica → `(error) READONLY You can't write against a read only replica.` Comportamiento default verificado.

**Fase 3 — leer `INFO replication` en ambos**:
- Master: `role:master`, `connected_slaves:1`, `slave0:ip=172.19.0.3,port=6379,state=online,offset=728,lag=0`, `master_replid:6a5f84a7…`, `master_repl_offset:728`.
- Réplica: `role:slave`, `master_host:redis`, `master_port:6379`, `master_link_status:up`, mismo `master_replid`, `master_repl_offset` casi idéntico (diferencia 14 bytes por keepalives entre las dos ejecuciones consecutivas del comando).

**Fase 4 — propagación en vivo**:
- Terminal A (master): `SET propagacion:test "funciona"`.
- Terminal B (réplica), inmediatamente después: `GET propagacion:test` → `"funciona"`. `KEYS propagacion:*` → `1) "propagacion:test"`.
- Propagación sub-milisegundo en red bridge Docker local.

**Fase 5 — matar el master y observar reacción de la réplica**:
- `docker compose stop redis` → master en `Exited`, réplica sigue `Up`.
- `INFO replication` en réplica tras ~23s: `master_link_status:down`, `master_last_io_seconds_ago:-1`, `master_link_down_since_seconds:23`. `role:slave` sin cambios. `master_replid:6a5f84a7…` preservado (por si el master vuelve pronto).
- `GET propagacion:test` en réplica → `"funciona"`. Lecturas siguen funcionando.
- `SET post:mortem "el master ha muerto"` en réplica → `(error) READONLY`. Escrituras siguen rechazadas.

**Fase 6 — reanimar el master y observar PSYNC2**:
- `docker compose start redis`.
- Master reiniciado: `role:master`, `connected_slaves:1` (reconexión automática), `master_replid:8e9595e17e…` (nuevo), **`master_replid2:6a5f84a7…` (el antiguo preservado)**, `master_repl_offset:2383`, `second_repl_offset:2384`, `repl_backlog_histlen:0` (backlog vacío, no ha hecho falta acumular nada).
- Réplica reconectada: `master_link_status:up`, mismo `master_replid` que el master reiniciado, `master_replid2` también coincide.
- `GET propagacion:test` en réplica → `"funciona"`. La clave sobrevivió al ciclo completo: master la persistió en RDB al SIGTERM, réplica no se rehidró (PSYNC evitó el full sync destructivo).
- **Predicción mía (Claude) errónea corregida en el momento**: había dicho "full sync porque el master arranca con offset 0". Realidad: PSYNC2 preserva identidad vía RDB. Registrado como corrección en la conversación.

**Fase 7 — gate del servicio tras profile `ha-lab`**:
- Añadido `profiles: - ha-lab` al servicio `redis-replica`.
- Verificación A: `docker compose up -d` (sin profile) → solo levanta `document-analyzer-redis`.
- Verificación B: `docker compose --profile ha-lab up -d` → levanta master + réplica.
- Gotcha encontrado en el proceso: `docker compose down` sin profile no baja la réplica que ya estaba corriendo, la deja huérfana. `--remove-orphans` **no** la limpia tampoco (bug conocido: issue #8432 y #11793 del repo docker/compose, servicios con profile añadido después de crear el contenedor quedan en limbo). Fix: `docker compose --profile ha-lab down`.

## Decisiones lockeadas

1. **Replicación ≠ HA en Redis puro**. Sin Sentinel o Cluster, master caído = servicio degradado (lecturas sí, escrituras no), sin promoción automática. En producción se monta capa de HA por encima.
2. **Réplica sin volumen** cuando se quiere reforzar "estado derivado". Rehidrata desde master vía full sync al arrancar. Alternativa (bind mount propio en la réplica) es válida pero conceptualmente peor para el modelo mental.
3. **Puerto mapping `HOST:CONTAINER`** — el orden es "de fuera hacia dentro". Puerto contenedor no cambia (Redis siempre en 6379 interno); mapeo al host desacopla acceso externo del puerto interno del proceso.
4. **DNS interno de Compose por nombre de servicio**. Contenedores en la misma red de Compose se resuelven por el nombre del servicio (`redis`, `redis-replica`). Puerto usado entre contenedores = puerto **interno** del proceso destino (no el mapeado al host).
5. **Profiles de Compose para servicios opcionales**. Servicios que no son parte del stack "por defecto" (labs, herramientas de debugging, componentes ha-lab, admin UIs) van gated tras un profile. Activación explícita con `--profile X` o `COMPOSE_PROFILES=X`.
6. **`--remove-orphans` NO limpia servicios con profile añadido después**. Bug conocido de Docker Compose. Regla: si trabajas con profiles, siempre activa todos los relevantes en `down` (o usa `COMPOSE_PROFILES=X,Y,Z docker compose down`).
7. **PSYNC2 preserva identidad del stream vía RDB** en shutdown limpio. Reiniciar el master con SIGTERM permite que las réplicas hagan PSYNC en lugar de full sync al reconectar. SIGKILL rompería esta garantía.

## Deuda / Gotchas al cerrar

1. **Contenedores del profile `ha-lab` quedan levantados al cerrar sesión**. Recomendación: `docker compose --profile ha-lab down` al terminar el día.
2. **Bug conocido de `--remove-orphans` con profiles**. Documentado en decisión #6. No es fix aplicable, es workaround (activar profile en down).
3. **Deuda técnica activa del arco Redis (heredada de sesiones anteriores, sin cambios)**:
   - Investigar bug `LlmLoggingAdvisor` × `MessageChatMemoryAdvisor` (duplicación en Redis cuando ambos activos). Probar cambio de `getOrder()` observando con `MONITOR`.
   - `MULTI/EXEC` para atomicidad de `saveAll` en `RedisChatMemoryRepository` vía `SessionCallback<>`.
   - Mover TTL de chat memory a `application.properties` como `document-analyzer.chat-memory.ttl-hours=24`.
   - Persistir `appendonly yes` en config permanente si algún momento se decide AOF permanente en el proyecto.
4. **Redis Sentinel y Redis Cluster no vistos en lab**. Nivel conceptual cubierto (qué son, qué problemas resuelven, cuándo elegir uno u otro). Montarlos en lab es tema propio de futuras sesiones, no bloquea nada del roadmap actual.

## Frases ⭐⭐⭐ interview-ready

**Replicación en 3 frases**:
> Replicación master-réplica en Redis: una instancia designada como master (única aceptando escrituras) y una o más réplicas manteniendo copia continua del dataset. Propagación **asíncrona**: el master responde OK al cliente antes de propagar; ventana típica de milisegundos, con implicación de que un failover puede perder las últimas escrituras confirmadas. Motivos habituales: read-scaling (repartir lecturas), alta disponibilidad (promoción de réplica ante caída del master), y backups en réplica sin castigar al master.

**Sync inicial vs sync continuo**:
> Al arrancar una réplica contra un master con datos, se hace **full sync**: master lanza BGSAVE con fork, envía el RDB completo a la réplica por TCP, y luego drena el replication backlog acumulado durante la transferencia. En régimen normal, el master reenvía cada comando de escritura por el socket abierto hacia cada réplica que lo aplica localmente. Ante desconexiones breves, si el offset de la réplica sigue dentro del `repl_backlog_size` del master, se hace **PSYNC** enviando solo el delta; si no, full sync completo.

**PSYNC2 y persistencia del stream**:
> Redis persiste `replid` y `master_repl_offset` dentro del RDB al hacer shutdown limpio. Al reiniciar el master, esos metadatos se recuperan y se exponen como `master_replid2` y `second_repl_offset` en `INFO replication`. Cuando las réplicas reconectan, el master reconoce el stream anterior y hace PSYNC en lugar de full sync. Es una optimización clave (PSYNC2) que evita full syncs innecesarios tras reinicios controlados del master.

**Replicación ≠ HA**:
> Replicación en Redis no es lo mismo que alta disponibilidad. La replicación master-réplica te da copias sincronizadas del dataset, pero el failover automático requiere **Redis Sentinel** o **Redis Cluster**. Sin esa capa por encima, una caída del master deja a las réplicas huérfanas: siguen sirviendo lecturas sobre el último estado conocido, pero permanecen en modo `slave` read-only rechazando escrituras hasta que alguien humano promocione una manualmente.

**Replicación vs sharding**:
> Son dos ejes ortogonales de escalabilidad. **Replicación** = N copias del mismo dataset, resuelve disponibilidad y read-scaling. **Sharding** = N particiones de un dataset dividido, resuelve escala de capacidad (RAM total, throughput de escritura, tamaño del keyspace). Redis Cluster combina ambos: divide el keyspace en 16384 hash slots repartidos entre múltiples masters, cada uno con sus propias réplicas.

**READONLY y prevención de divergencia**:
> Las réplicas de Redis arrancan por defecto en modo read-only (`replica-read-only yes`) rechazando escrituras con error `READONLY`. Es un safeguard para prevenir divergencia: si permitieras escrituras locales en la réplica, tendrías dos datasets que nunca reconcilian — la escritura local no sube al master, y cuando el master propague sus propios cambios, la réplica tendrá estado incoherente. Existe flag para desactivarlo (`replica-read-only no`) pero se reserva a casos muy específicos.

## Estado de la app y roadmap

- `document-analyzer-ai` funcionalmente sin cambios en código Java. Solo `docker-compose.yml` tocado.
- Bloque 4 Redis: 2/3 subtemas cerrados (RDB/AOF, replicación master-réplica). **Pendiente**: cache-aside pattern (próxima sesión del arco, requiere código Java nuevo).
- Sentinel y Cluster cubiertos conceptualmente pero no en lab. Suficiente para entrevistas nivel junior/mid; montarlos queda parkeado para futuras profundizaciones si algún día se retoma.
- Roadmap post-Git/Bash oficial (AWS → Kubernetes → OAuth2 → Observability) no tocado. AWS Sesión 11 (Terraform remote state con S3+DynamoDB) sigue siendo el siguiente hito si se decide priorizar sobre cache-aside.

## Correcciones y aprendizajes de proceso

- **Corrección mía (Claude) en directo**: predije "full sync tras reinicio del master porque arranca con offset 0". Realidad: PSYNC2 preserva identidad vía RDB, se hace PSYNC. Corregido y explicado en el momento. Aprendizaje: no simplificar excesivamente en las predicciones sin verificar el mecanismo real, especialmente cuando la implementación tiene optimizaciones no obvias.
- **Descubrimiento de bug de Docker Compose sobre `--remove-orphans` + profiles**: no era problema de la configuración, es limbo conocido del daemon cuando se añade un profile a un servicio ya corriendo. Documentado en decisión lockeada #6.
- **Duplicación de un mensaje del usuario en la conversación**: Tole reenvió por error el mismo mensaje dos veces (el que contenía la salida del `INFO replication` post-reinicio del master). Detectado y evitado responder dos veces al mismo output.

## Próxima sesión

Opciones ordenadas por continuidad:

1. **Bloque 4 parte 3**: cache-aside pattern en `document-analyzer-ai` (cierra el arco Redis). Requiere código Java nuevo: elegir endpoint objetivo, wrapear con lógica lookup → miss → DB → poblar caché → return, testear invalidación. Duración estimada 90-120 min.
2. **AWS Sesión 11**: Terraform remote state con S3 + DynamoDB (continuación de la 10). Hands-on, cabe en 2h. Rompe con Redis pero avanza el roadmap post-Git/Bash.
3. Investigar bug `LlmLoggingAdvisor` × `MessageChatMemoryAdvisor` con `MONITOR` y `getOrder()`. 45-60 min.
4. Mini-sesión de reactivación de algún tema oxidado (JWT refresh, JPA, Testing, Resilience4j, etc.) — Q&A socrático con código delante.

Ver también `PromptContinuacion-Redis-Bloque4-CacheAside-2026-08-15.md` cuando se genere (aún no creado — pendiente de decisión sobre siguiente sesión).
