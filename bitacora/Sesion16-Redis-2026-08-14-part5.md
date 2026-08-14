# Sesión Redis — 2026-08-14 (part 5)

**Bloque 4 (opcional del arco Redis) — Parte 1 de N: Persistencia RDB y AOF.**

Duración efectiva: ~17:00–19:30, viernes tarde. Energía media, buen ritmo sostenido.

Contexto de sesión: se abrió el bloque 4 con la idea de cubrir RDB/AOF + replicación master-replica + cache-aside en 2h. Scope reducido honestamente al arrancar: solo persistencia (RDB + AOF) esta sesión, resto para siguientes. Decisión correcta a posteriori — cubrir RDB+AOF con lab empírico completo consumió las 2h enteras.

## Repo state al cerrar

- Rama `main`, HEAD en un commit nuevo por delante del `4300159` con el que arrancó la sesión.
- Working tree clean.
- Contenedor Redis parado (`docker compose down`).
- Directorio `redis-data/` intacto con ficheros de la sesión: `dump.rdb` (195 bytes, timestamp 19:16) y `appendonlydir/` con los tres ficheros AOF de generación 2 (huérfanos, ver "Deuda / Gotchas" abajo).

## Commits generados en la sesión

1. `refactor(observability): rename coste_usd to costUsd for naming consistency` — cambio parked de la sesión anterior, commiteado al arrancar para dejar working tree limpio antes del lab.
2. `feat(persistence): enable Redis persistence via bind mount` — añade bind mount `./redis-data:/data` al servicio `redis` del `docker-compose.yml` y excluye `redis-data/` del `.gitignore`. Verificado empíricamente que RDB sobrevive a ciclos `down/up`.

## Conceptos cubiertos

### RDB — Redis Database Backup

- **Qué**: snapshot binario del estado completo de la RAM, escrito a `dump.rdb`. Volcado completo, no incremental.
- **Cuándo se dispara (4 formas)**:
  1. `SAVE` — manual síncrono. Bloquea el servidor. No usar en producción.
  2. `BGSAVE` — manual asíncrono. Fork del proceso, hijo escribe, padre sigue atendiendo. Uso normal.
  3. Política automática vía directiva `save <segundos> <cambios>` en `redis.conf`. Múltiples reglas en OR.
  4. Al shutdown limpio (SIGTERM), Redis hace un save final antes de morir.
- **Asimetría SIGTERM vs SIGKILL** (crítica): SIGTERM es interceptable, permite cleanup y save. SIGKILL (OOM killer, `kill -9`, hardware pull) no. Con SIGKILL pierdes todo lo posterior al último snapshot.
- **Ventana de pérdida** = intervalo entre snapshots. Con política default (`3600 1 300 100 60 10000`), puede llegar a **1 hora** en escenarios de baja actividad.
- **Ventajas**: fichero compacto (binario optimizado), restore rápido (leer bytes vs replay), bajo overhead runtime.
- **Uso típico**: backups, disaster recovery, casos que toleran pérdida de segundos a minutos.

### AOF — Append Only File

- **Qué**: log secuencial de cada operación de escritura (SET, LPUSH, DEL, EXPIRE, etc.). Formato RESP crudo. Analogía: WAL de PostgreSQL, oplog de MongoDB.
- **Restore**: replay del log al arranque. Ejecuta cada comando en orden contra una BD vacía, reconstruyendo el estado.
- **Trade-off vs RDB**: fichero más grande (verbose), restore más lento (replay > carga binaria), pero **ventana de pérdida mucho menor**.
- **Directiva `appendfsync` (tres valores)**:
  - `always`: fsync en cada operación antes de responder OK al cliente. Cero pérdida, rendimiento cae 10-100x. Casi nadie lo usa.
  - `everysec`: DEFAULT. Fsync una vez por segundo en hilo background. Ventana de pérdida máxima 1s, rendimiento casi óptimo.
  - `no`: sin fsync explícito, delega al kernel. Ventana ~30s. Raro.
- **AOF rewrite** (`BGREWRITEAOF`): condensa el log — en lugar de mantener 1000 `INCR contador`, guarda `SET contador 1000`. Se dispara manualmente o por política automática.

### Multi-Part AOF (Redis 7+)

Estructura moderna que reemplazó el `appendonly.aof` monolítico de Redis ≤6:

- `appendonly.aof.<N>.base.rdb`: snapshot base en **formato RDB binario** (más compacto que AOF verbose). Congelado en el momento del último rewrite.
- `appendonly.aof.<N>.incr.aof`: fichero incremental donde se appendean las nuevas operaciones desde el último rewrite. Formato RESP.
- `appendonly.aof.manifest`: metadata que indica al Redis al arrancar qué ficheros cargar y en qué orden.

Al arrancar: manifest → base.rdb (rápido) → incr.aof (replay lento pero corto).

Con cada `BGREWRITEAOF`: se genera un nuevo `.base.rdb` con el estado actual, se abre un `.incr.aof` vacío nuevo, se rota el numerito (1 → 2 → 3...), se eliminan los ficheros de la generación anterior atómicamente.

### Convivencia RDB + AOF

En producción real, **ambos activos simultáneamente** es la config estándar:
- AOF cubre la baja ventana de pérdida (~1s).
- RDB proporciona backup portable y restore rápido.
- Al arrancar con ambos ficheros presentes, **Redis prioriza AOF** para el restore (más granular, más reciente).

Excepción: casos donde la durabilidad no importa (cache-aside puro con PostgreSQL como fuente de verdad) — ahí solo RDB o ninguno.

## Laboratorio empírico (ejecutado)

**Fase 1 — demostrar el problema sin volumen**:
- SET clave → `docker compose down` (elimina contenedor) → `up` → GET devuelve `(nil)`.
- Aunque Redis hizo "Saving the final RDB snapshot before exiting" al SIGTERM, el filesystem interno del contenedor murió con él. Persistencia Redis y persistencia Docker son dos capas independientes.

**Fase 2 — añadir bind mount y verificar persistencia**:
- Bind mount `./redis-data:/data` en `docker-compose.yml`.
- Añadido `redis-data/` al `.gitignore` (regla general: **runtime state fuera de git, código y config dentro**).
- Repetido experimento: SET → `SAVE` manual → `down` → `up` → GET devuelve valor correcto. Persistencia empíricamente confirmada.

**Fase 3 — política automática**:
- `CONFIG GET save` mostró default: `"3600 1 300 100 60 10000"` (tres reglas OR).
- `CONFIG SET save "10 1"` para ver save auto en vivo. Timestamp del `dump.rdb` cambió sin comando manual, confirmando el trigger.

**Fase 4 — activar AOF y ver estructura Multi-Part**:
- `CONFIG SET appendonly yes` creó `appendonlydir/` con `base.rdb` + `incr.aof` (vacío) + `manifest`.
- Tres SET nuevos → `incr.aof` creció a 145 bytes.
- `cat appendonly.aof.1.incr.aof` mostró formato RESP crudo (`*3\n$3\nSET\n$10\naof:test:1\n$4\nhola\n...`).

**Fase 5 — verificar rotación con BGREWRITEAOF**:
- `BGREWRITEAOF` → generación pasó de 1 a 2. `.1.*` eliminados atómicamente, `.2.base.rdb` (195 bytes, incluye las 3 claves nuevas) y `.2.incr.aof` vacío aparecieron.

**Fase 6 — descubrimiento accidental: efimeridad de `CONFIG SET`**:
- `docker compose restart redis` → todas las claves sobrevivieron (RDB hizo el rescue al SIGTERM), pero `CONFIG GET appendonly` devolvió `no`. AOF quedó desactivado silenciosamente porque el `redis.conf` default no persistió el cambio hecho con `CONFIG SET`.
- Los ficheros AOF en `appendonlydir/` quedaron **huérfanos**: presentes en disco, pero Redis no los toca porque `appendonly no`. Estado inconsistente peligroso.

## Decisiones lockeadas

1. **`CONFIG SET` es efímero**. Vive solo en memoria del proceso Redis. En producción, la configuración de persistencia **siempre** va en `redis.conf` versionado o en el ConfigMap del orquestador. `CONFIG SET` solo para experimentos, troubleshooting o cambios que se persisten inmediatamente después (`CONFIG REWRITE` o edición de fichero).
2. **Runtime state fuera de git**. `dump.rdb`, `appendonlydir/`, cualquier fichero que la BD genere en runtime → nunca commitear. Aplicable a Redis, PostgreSQL, MongoDB, cualquier data store.
3. **`.gitignore` para directorios usa `nombre/` (barra final, sin barra inicial)**. Sin barra final es ambiguo (fichero o directorio); con barra inicial se ancla a la raíz (no escala si el patrón podría aparecer anidado). Regla: `redis-data/` correcto, `./redis-data` incorrecto.
4. **`git check-ignore -v <path>` para verificar patrones .gitignore** antes de asumir que funcionan. Silencio = no matchea. Detalle: si el path no existe en disco todavía, hay que consultarlo con barra explícita (`redis-data/`) para que git lo trate como directorio.
5. **En producción real: RDB + AOF ambos activos**. AOF con `appendfsync everysec` (default) es el sweet spot durabilidad/rendimiento. RDB como backup portable.

## Deuda / Gotchas al cerrar

1. **Ficheros AOF huérfanos en `redis-data/appendonlydir/`**. `appendonly no` en la config actual del contenedor, pero los ficheros de generación 2 siguen en disco. Si mañana se reactiva AOF con `CONFIG SET`, Redis regenerará el base desde cero (los antiguos no aportarán datos). No es bug, es estado esperado de la sesión.
2. **Bind mount con ownership `999:tole`**. UID 999 = usuario `redis` del contenedor. Requiere `sudo` para leer/eliminar desde WSL. Si algún día se quiere limpiar el directorio, `sudo rm -rf redis-data/`.
3. **`docker-compose.yml` usa bind mount, no named volume**. Decisión pedagógica para ver los ficheros directamente desde WSL. En producción real casi siempre named volume o volumen gestionado por el orquestador (K8s PersistentVolumeClaim).
4. **Deuda técnica activa del arco Redis (heredada, no arrancada en esta sesión)**:
   - Investigar bug `LlmLoggingAdvisor` × `MessageChatMemoryAdvisor` (duplicación de mensajes cuando ambos activos).
   - `MULTI/EXEC` para atomicidad de `saveAll` en `RedisChatMemoryRepository`.
   - Mover TTL de chat memory a `application.properties`.

## Frases ⭐⭐⭐ interview-ready

**RDB en 3 frases**:
> RDB es snapshotting periódico: Redis vuelca la RAM a un fichero binario (`dump.rdb`) en momentos disparados por política automática, comando manual (SAVE/BGSAVE) o shutdown limpio. Trade-off principal: rápido de restaurar y compacto en disco, pero **ventana de pérdida** entre snapshots — si el proceso muere abruptamente (SIGKILL, OOM, hardware), pierdes todo lo posterior al último snapshot. Uso típico: backups y disaster recovery; **insuficiente en solitario** para casos que no toleran pérdida.

**AOF en 3 frases**:
> AOF es logging incremental: cada operación de escritura se appendea a un fichero de log en formato RESP. Al arrancar, Redis **replaya** el log para reconstruir el estado. Trade-off central: mayor durabilidad que RDB (ventana configurable con `appendfsync`: 0s con `always`, 1s con `everysec` default, ~30s con `no`) a cambio de ficheros más grandes y restore más lento. **AOF rewrite** (manual con `BGREWRITEAOF` o automático) evita crecimiento indefinido consolidando el log en un base compacto.

**Config de persistencia en producción**:
> En producción típica de banca: RDB + AOF ambos activos. AOF con `appendfsync everysec` para ventana de pérdida de 1 segundo con rendimiento casi óptimo. RDB para backup portable y disaster recovery. La configuración **siempre** vía `redis.conf` versionado o ConfigMap del orquestador, nunca solo con `CONFIG SET` que es efímero al restart.

**Multi-Part AOF (Redis 7+)**:
> En Redis 7 el AOF pasó a ser Multi-Part: en lugar de un único `appendonly.aof`, se organiza como un directorio con `base.rdb` (snapshot base en formato RDB), `incr.aof` (log incremental en RESP) y `manifest` (índice). Beneficios: restore más rápido (base es binario), rewrite atómico (el nuevo base se escribe separado y solo al terminar se actualiza el manifest), estructura más manejable operacionalmente.

## Estado de la app y roadmap

- `document-analyzer-ai` funcionalmente sin cambios en código Java. Solo `docker-compose.yml` + `.gitignore` tocados.
- Bloque 4 Redis: 1/3 subtemas cerrados (RDB/AOF). Pendientes: replicación master-replica + cache-aside pattern.
- Roadmap post-Git/Bash oficial (AWS → Kubernetes → OAuth2 → Observability) no tocado en esta sesión. Reanudable cuando cierre el bloque 4 opcional o antes si se decide priorizar.

## Próxima sesión

Ver `PromptContinuacion-Redis-Bloque4-Replicacion-2026-08-14.md`.
