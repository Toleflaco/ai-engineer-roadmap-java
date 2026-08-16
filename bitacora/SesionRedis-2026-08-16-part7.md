# Sesión Redis — 2026-08-16 (part 7)

**Bloque 4 (opcional del arco Redis) — Parte 3 de N: Cache-aside pattern en `document-analyzer-ai`.**

Duración efectiva: ~3h, domingo mañana. Energía alta (buen descanso), ritmo sostenido. Sesión larga pero productiva: teoría + diseño de componente + implementación + validación empírica exhaustiva + commit.

Contexto de sesión: continuación directa del bloque 4 tras la parte 2 (replicación master-réplica de ayer). Scope acotado desde el arranque: solo cache-aside sobre el endpoint `POST /analyze`. `/analyze/pdf` diferido a follow-up autónomo (Tole lo hará solo aplicando el mismo patrón). Decisión correcta a posteriori — la implementación con Sócrates puro + validación con MONITOR + validación fail-safe consumieron las 3h enteras.

## Repo state al cerrar

- Rama `main`, HEAD un commit por delante del `db4ff84` con el que arrancó la sesión. Sincronizado con `origin/main`.
- Working tree clean.
- Contenedor `document-analyzer-redis` levantado al cerrar (sin profile, master solo — la réplica no se necesita para cache-aside). Recomendación: `docker compose down` al terminar el día.
- Redis flusheado con `FLUSHALL` antes del commit para dejar el estado neutral.
- App Spring Boot parada.

## Commits generados en la sesión

1. `feat(analyze): add cache-aside for /analyze using Redis` — introduce `CvAnalysisCache` en el package `analyze/`, modifica `AnalyzeController` para orquestar el patrón (get → return temprano si hit; miss → LLM → put → return). Mensaje bilingüe extenso documentando decisiones de diseño (SHA-256, atomicidad SET+TTL, fail-safe) y validaciones empíricas (MONITOR, tiempos, fail-safe con Redis parado, recovery automático tras arranque).

## Conceptos cubiertos

### Cache-aside — modelo mental base

- **Qué**: patrón de caché "al lado" del flujo BD. Aplicación orquesta explícitamente las dos capas (caché + fuente de verdad). Caché es **pasiva**: no se rellena sola, no invalida sola, no sabe nada de la BD.
- **Flujo canónico de lectura**:
  1. App consulta caché.
  2. HIT → devuelve directamente.
  3. MISS → consulta BD/servicio caro, obtiene dato, lo guarda en caché, lo devuelve.
- Contrasta con patrones "en medio" (write-through, write-behind, read-through) que requieren infraestructura más compleja. Cache-aside es el default por simplicidad y control explícito.
- **Ventajas**: implementación simple, tolerante a fallos de caché (Redis muere → todo miss → BD directa), memoria eficiente (solo se cachea lo pedido).
- **Desventajas**: primera petición siempre lenta, coherencia caché↔BD es responsabilidad del código, susceptible a cache stampede en claves populares.

### Cache stampede (thundering herd) — mencionado, no implementado

- Cuando una clave popular expira, N peticiones simultáneas hacen miss y bombardean la BD/LLM a la vez.
- Mitigaciones estándar (no cubiertas en lab): locks distribuidos (singleflight), refresh anticipado antes de expirar, jitter en TTLs para desincronizar expiraciones.
- Nivel conceptual suficiente para entrevista: "el problema existe, se mitiga con locks, refresh anticipado o jitter en TTLs". Implementarlo es tema propio.

### Invalidación en writes: DELETE, no UPDATE

- Escenario canónico de race condition entre writers concurrentes:
  1. Writer A escribe BD → 100€.
  2. Writer B escribe BD → 200€ (BD queda en 200€, correcto).
  3. B escribe caché → 200€.
  4. A escribe caché → 100€ (caché queda en 100€ stale, incoherente con BD).
- Estrategia **DELETE**: mismo escenario pero 3-4 son `DEL` idempotentes. La siguiente lectura hace miss y repuebla con el valor fresco de la BD.
- Regla que va a entrevista, palabra clave incluida: "invalidate on write" via delete, no via update.
- Existe race residual más rara (READ old + WRITE new + POPULATE cache with old, entre líneas). Soluciones avanzadas: double-delete diferido, versionado por timestamp. Tema para senior.

### SHA-256 sobre texto normalizado como cache key

- **Por qué SHA-256 y no `hashCode()`**: `hashCode()` de Java = 32 bits, colisiones habituales, no criptográfico. Riesgo real de servir análisis cruzado entre CVs distintos. SHA-256 = 256 bits, colisión práctica imposible, determinista, longitud fija.
- **Formato de clave con namespacing jerárquico**: `analyze:cv:<sha256hex>`. Convención Redis idiomática (mismo estilo que `chat:memory:<id>` en el proyecto).
- **Normalización mínima antes del hash**: `text.trim()`. Sin normalizar, un espacio extra al final produce hash distinto → miss falso. Sobre-normalizar (colapsar whitespace interno, lowercase, etc.) es trade-off: dos CVs sutilmente distintos podrían colisionar. Para lab, trim() suficiente.
- **Detalles técnicos**:
  - `MessageDigest.getInstance("SHA-256")` — SHA-256 garantizado en toda JVM, `NoSuchAlgorithmException` es checked pero nunca ocurre en la práctica. Envolver en `IllegalStateException("SHA-256 not available", e)` internamente.
  - **`getBytes(StandardCharsets.UTF_8)` siempre explícito, nunca `getBytes()` a secas**: default charset varía por sistema (Windows CP1252 vs Linux UTF-8 → hashes distintos para el mismo texto). Bug clásico en producción.
  - Conversión bytes → hex con **`HexFormat.of().formatHex(bytes)`** (Java 17+). Alternativa antigua: loop con `String.format("%02x", b)`. `BigInteger` es feo, evitar.

### Spring Data Redis — operaciones agrupadas por tipo

- `RedisTemplate` / `StringRedisTemplate` expone operaciones **agrupadas por tipo de dato Redis**, no comandos planos.
- `opsForValue()` → `ValueOperations` → `GET`, `SET`, `INCR`, `DECR`, `SETEX`.
- `opsForList()` → `ListOperations` → `RPUSH`, `LPUSH`, `LRANGE`.
- `opsForHash()` → `HashOperations` → `HSET`, `HGET`, `HGETALL`.
- `opsForSet()` → `SetOperations` → `SADD`, `SMEMBERS`, `SINTER`.
- `opsForZSet()` → `ZSetOperations` → sorted sets.
- Cada uno devuelve una interfaz especializada con solo los comandos aplicables a esa estructura, mapeados internamente al comando Redis subyacente.
- En el proyecto: `RedisChatMemoryRepository` usa `opsForList()` (conversaciones = secuencias); `CvAnalysisCache` usa `opsForValue()` (análisis = valor único por CV).

### Atomicidad de SET+TTL en un solo comando

- `redis.opsForValue().set(key, value, TTL)` traduce a `SET key value EX <segundos>` — **un solo comando atómico**.
- Alternativa mala: `set(key, value)` + `expire(key, TTL)` como dos comandos separados. Ventana de vulnerabilidad: si el proceso muere entre los dos comandos, la clave queda **sin TTL** (leak permanente hasta rotación manual o eviction por memoria llena).
- **Regla en Redis**: siempre que exista la variante atómica (SET con EX/PX, o `SET NX EX`), usarla en vez de comandos separados.
- Verificado en MONITOR del lab: `SET "analyze:cv:..." "{json}" "EX" "86400"` — un solo comando, la clave nace con TTL de 24h.

### Fail-safe / cache degradation

- **Filosofía**: Redis no es fuente de verdad; una petición nunca debe fallar por fallo de infraestructura de caché. Preferir respuesta lenta a respuesta fallida.
- **Implementación**: `try/catch (Exception e)` alrededor de cada operación Redis.
  - `get` falla → log WARN + return `Optional.empty()` (el caller lo trata como miss, sigue al backend).
  - `put` falla → log WARN + swallow (el `CvSummary` ya está computado y se devuelve al cliente; solo se pierde la oportunidad de cachearlo).
- **Nivel de log**: WARN, no ERROR. Regla: ERROR se reserva a lo que rompe la petición del cliente; la caché no rompe la petición por diseño. Un dashboard bien montado alerta cuando el volumen de WARN de caché cruza un umbral.
- **Contenido del log**: mensaje + clave (para debugging) + excepción completa (para stack trace en post-mortem).
- **Anti-patrón evitado**: log-and-throw. Si logeas Y relanzas, capas superiores loguearán otra vez → 3 líneas de log por 1 evento. Regla: **o logueas y tragas, o dejas propagar sin loguear**.
- **`catch (Exception e)` genérico vs específico**: para lab, `Exception` cubre todo el fail-safe de una vez. En código de producción se prefiere capturar excepciones específicas (`DataAccessException` de Spring Data, `JsonProcessingException` de Jackson, etc.).

### Separación de responsabilidades — componente vs controller

- Podría meterse toda la lógica de caché en el controller. Anti-patrón: controller que hashea, serializa JSON, habla con Redis, renderiza prompts y llama al LLM viola SoC.
- Extraer a componente propio (`CvAnalysisCache`):
  - **Testeable en aislamiento** (mock de `StringRedisTemplate`, tests unitarios del hash y round-trip JSON).
  - Refleja arquitectura idiomática: caché es capa de infraestructura, no de presentación.
  - En proyecto real sería interface + implementación concreta (`RedisCvAnalysisCache`) para swappear backends (in-memory en tests, distintos backends). Simplificado a clase directa en lab.
- API pública del componente: `Optional<CvSummary> get(String cvText)` + `void put(String cvText, CvSummary summary)`. El controller no sabe de SHA-256, prefijo, TTL, JSON.
- **`get` + `put` explícitos vs `getOrCompute(key, supplier)`**: la versión funcional es más compacta pero esconde la estructura del patrón. Para el lab, forma explícita con `if hit → return early` para que la lógica cache-aside quede en el código del controller. Refactor a `getOrCompute` es trivial después.

### Lettuce y reconexión automática

- **Lettuce** = cliente Redis por defecto de Spring Data Redis (desde Spring Boot 2.0).
- `ConnectionWatchdog` de Lettuce reintenta reconectar automáticamente cuando detecta que la conexión se rompe.
- Verificado empíricamente: `docker compose stop redis` → WARN de `ConnectionWatchdog` cada pocos segundos. `docker compose start redis` → reconexión automática, la app **no necesita reinicio** para volver a usar la caché.
- En producción esto es clave: reinicios/failovers de Redis son transparentes para la aplicación siempre que Lettuce pueda reconectar.

## Laboratorio empírico (ejecutado)

**Fase 1 — diseño y creación de `CvAnalysisCache`**:
- Fichero nuevo `src/main/java/dev/toleflaco/document_analyzer_ai/analyze/CvAnalysisCache.java`.
- `@Component`, inyección por constructor de `StringRedisTemplate` + `ObjectMapper` (Jackson 3, import `tools.jackson.databind.ObjectMapper` NO `com.fasterxml.jackson`, coherencia con `RedisChatMemoryRepository`).
- Constantes: `KEY_PREFIX = "analyze:cv:"`, `TTL = Duration.ofHours(24)`.
- Métodos:
  - `Optional<CvSummary> get(String cvText)`: `buildKey` → `opsForValue().get(key)` → si null return empty; si no, `objectMapper.readValue(json, CvSummary.class)` → return Optional.of.
  - `void put(String cvText, CvSummary summary)`: `buildKey` → `writeValueAsString` → `opsForValue().set(key, json, TTL)`.
  - `private String buildKey(String cvText)`: `KEY_PREFIX + sha256Hex(cvText.trim())`.
  - `private static String sha256Hex(String input)`: `MessageDigest` + UTF-8 explícito + `HexFormat.of().formatHex(bytes)`, envuelto en `IllegalStateException` interno para el `NoSuchAlgorithmException` checked.
- Ambos `get` y `put` con `try/catch (Exception e)` + `log.warn` con clave + stack trace.

**Fase 2 — modificación de `AnalyzeController`**:
- Inyección por constructor con nuevo parámetro `CvAnalysisCache cache`, campo `final`.
- Reformateo del constructor con 4 parámetros uno por línea (convención Spring).
- Método `analyze()`:
  ```
  Optional<CvSummary> cached = cache.get(request.cv());
  if (cached.isPresent()) return cached.get();
  // ... lógica LLM existente sin tocar ...
  CvSummary summary = converter.convert(rawResponse);
  cache.put(request.cv(), summary);
  return summary;
  ```
- Cirugía mínima: no se toca prompt, DTO de request, `CvSummary`, ni `RedisChatMemoryRepository`.

**Fase 3 — verificación en vivo con MONITOR**:
- Setup: 3 terminales (app Spring Boot, `redis-cli MONITOR`, `curl`).
- `FLUSHALL` previo para estado limpio y evitar confusión de tiempos.
- **Miss #1** (CV nuevo): MONITOR muestra `GET analyze:cv:ab6445...` (null) → 3.7s después `SET analyze:cv:ab6445... {json} EX 86400`. `curl` reporta 3.16s.
- **Hit** (mismo CV): MONITOR muestra solo `GET` (sin SET). `curl` reporta 0.006s. Aceleración ~500x.
- **Miss #2** (CV modificado: "9 years" → "8 years"): hash distinto `87e0074b...`, mismo patrón GET(null)+SET, `curl` 3.33s.
- Verificado que `SET` incluye `EX 86400` = 24h en un solo comando atómico. TTL nace con la clave.

**Fase 4 — verificación de fail-safe con Redis caído**:
- `docker compose stop redis` con la app corriendo.
- Lanzada petición con CV nuevo:
  - **Log observado**:
    - `WARN ConnectionWatchdog: Cannot reconnect to [localhost:6379]: Connection refused` (Lettuce infra).
    - `WARN CvAnalysisCache: Redis GET failed for key analyze:cv:4c2e61e9..., treating as cache miss` + full stack trace `RedisSystemException → RedisException → SocketException: Connection reset`.
    - `INFO LlmLoggingAdvisor: llm call completed latency_ms=2806 tokens_in=827 tokens_out=161 cost_usd=0.004896`.
    - `WARN CvAnalysisCache: Redis PUT failed for key analyze:cv:4c2e61e9..., cache not populated` + stack trace.
  - **curl**: 200 OK con `CvSummary` válido, tiempo 2.82s (dominado por el LLM; la parte Redis fallida no añade latencia perceptible porque falla rápido).
- Comportamiento verificado: cache-aside degrada, no se rompe. WARN identifica claramente el evento sin contaminar alertas ERROR.

**Fase 5 — recovery automático de Lettuce**:
- `docker compose start redis` sin reiniciar la app.
- Lanzadas dos peticiones idénticas con CV "Test recovery":
  - **Primera**: 1.77s (miss real; Redis arrancó vacío tras cargar RDB que solo contenía claves de tests anteriores).
  - **Segunda**: 0.004s (hit).
- Confirmado que Lettuce reconectó automáticamente, sin necesidad de reiniciar la app.

**Fase 6 — descubrimiento colateral: persistencia RDB en acción**:
- Tras el recovery, `KEYS analyze:cv:*` mostró 3 claves de tests anteriores (`ab6445...`, `0347...`, `f24d...`).
- Origen: `docker compose stop` = SIGTERM al proceso Redis → shutdown limpio → `dump.rdb` final escrito con las claves activas → al arrancar de nuevo cargó ese RDB. Bind mount `./redis-data:/data` (heredado de la parte 1 del bloque 4) permite que el RDB sobreviva al ciclo `down`/`up`.
- Cierra el círculo con la parte 1 del bloque 4: persistencia y caché coexisten sin que la app tenga que saberlo.

**Fase 7 — cleanup y commit**:
- `FLUSHALL` para estado neutral.
- `git status` + `git diff` verificados: solo 2 ficheros esperados (`CvAnalysisCache.java` nuevo, `AnalyzeController.java` modificado).
- Commit bilingüe con parte inglesa + `---` + parte española sin tildes/ñ.

## Decisiones lockeadas

1. **Cache-aside es el patrón default de caché en Redis**. Simple, explícito, tolerante a fallos, control total desde el código de la app. Write-through / read-through / write-behind se reservan a casos específicos con infraestructura de soporte.
2. **Invalidación en writes vía DELETE, nunca UPDATE**. Colapsa races entre writers concurrentes; delete es idempotente. Nombre técnico del patrón: "invalidate on write".
3. **SHA-256 sobre texto normalizado como cache key**. `hashCode()` de Java NUNCA como key criptográfica; colisiones probables → riesgo real de servir contenido cruzado entre entidades.
4. **Encoding explícito UTF-8 en todo hashing**: `getBytes(StandardCharsets.UTF_8)`, jamás `getBytes()` a secas. Regla general aplicable a cualquier hashing o serialización binaria de strings.
5. **Atomicidad en operaciones Redis con TTL**: usar `SET key value EX seconds` (un comando) en vez de `SET` + `EXPIRE` separados (dos comandos con ventana de leak). Aplicable a cualquier operación donde Redis ofrezca variante atómica.
6. **Fail-safe / cache degradation es la estrategia default para caché sobre datos derivados**. Redis no es fuente de verdad → una petición nunca falla por caída de caché. WARN + return miss + seguir al backend.
7. **Log level WARN para fallos de caché, nunca ERROR**. ERROR se reserva a fallos que rompen la petición del cliente. Filosofía coherente con la anterior: la caché no rompe la petición por diseño.
8. **Anti-patrón log-and-throw**: o logueas y tragas, o dejas propagar sin loguear. Nunca ambos, evita logs duplicados en capas superiores.
9. **Caché como componente propio, no lógica inline en controller**. Separación de responsabilidades, testabilidad, sustituibilidad. Api pública mínima (`get`/`put`) que oculta hashing, serialización, TTL, prefijo.
10. **Jackson en este proyecto = Jackson 3 (`tools.jackson.databind.ObjectMapper`)**. Nunca importar `com.fasterxml.jackson.databind.ObjectMapper` aunque exista en el classpath por retrocompatibilidad transicional de Spring Boot 4.1. Coherencia con `RedisChatMemoryRepository`.

## Deuda / Gotchas al cerrar

1. **`/analyze/pdf` sin caché**. Diferido a follow-up autónomo de Tole: aplicar el mismo patrón pero calculando SHA-256 sobre bytes del PDF (`MultipartFile.getBytes()`), con cuidado del `InputStream` (one-shot: si lees para hashear, no puedes releer para enviar al LLM → bufferizar bytes antes). Cachear `CvSummary`, no el PDF entero.
2. **Cache stampede no mitigado**. Aceptable para un endpoint de análisis de CVs con carga baja. Si en producción hubiera claves populares con expiración simultánea, implementar singleflight con `SET NX EX` como lock distribuido.
3. **Sin invalidación explícita porque no hay endpoint de UPDATE de CV**. El TTL de 24h cubre el escenario natural (el mismo CV no se reanaliza más allá de un día). Si en el futuro se añade endpoint para "reanalizar forzado", habría que exponer `cache.delete(cv)` público y llamarlo desde ahí.
4. **`catch (Exception e)` demasiado genérico para código de producción**. Cubre todo el fail-safe de una vez, pero refactor recomendado a excepciones específicas (`DataAccessException`, `JsonProcessingException`) para poder distinguirlas en dashboards o para tratamiento diferenciado si algún día lo hubiera.
5. **Cache vacío no discriminado**. Un CV muy corto ("Test recovery") produce `CvSummary` vacío y también se cachea. Si esto es problema, validación en la capa de request (`@Valid` + `@Size(min=...)` en `AnalyzeRequest.cv()`) o filtro en `put` que no cachee si `summary` es semánticamente vacío. Fuera del alcance del bloque 4.
6. **Deuda técnica activa del arco Redis (heredada, sin cambios)**:
   - Investigar bug `LlmLoggingAdvisor` × `MessageChatMemoryAdvisor` (duplicación en Redis cuando ambos activos en `/chat`). Probar cambio de `getOrder()` observando con `MONITOR`.
   - `MULTI/EXEC` para atomicidad de `saveAll` en `RedisChatMemoryRepository` vía `SessionCallback<>`.
   - Mover TTL de chat memory a `application.properties` como `document-analyzer.chat-memory.ttl-hours=24`. Aplicar la misma refactorización al TTL de `CvAnalysisCache` cuando toque.
7. **Contenedor Redis levantado al cerrar sesión**. Recomendación: `docker compose down` al terminar el día.

## Frases ⭐⭐⭐ interview-ready

**Cache-aside en 3 frases**:
> Cache-aside es el patrón donde la aplicación orquesta explícitamente caché y BD. En lectura: consultar caché → si HIT devolver, si MISS ir a BD, poblar caché con el resultado, devolver. En escritura: escribir en BD e invalidar (**DELETE**, no UPDATE) la entrada de caché. La caché es pasiva: no sabe nada de la BD, no se rellena sola, no se invalida sola. Toda la coherencia es responsabilidad del código.

**Por qué DELETE y no UPDATE en la invalidación**:
> Con dos writers concurrentes, el orden de operaciones puede intercalarse dejando la caché con un valor stale respecto a la BD: writer A escribe BD 100€, writer B escribe BD 200€, B actualiza caché a 200€, A actualiza caché a 100€, y la caché queda incoherente. Con DELETE, ambas invalidaciones son idempotentes y la siguiente lectura repuebla desde la fuente de verdad. Es el patrón "invalidate on write".

**SHA-256 para claves de caché sobre contenido**:
> Nunca uses `hashCode()` de Java como cache key derivada de contenido: 32 bits, colisiones habituales, riesgo real de servir el contenido cacheado de otra entidad. SHA-256 sobre el contenido normalizado (con `trim()` mínimo, encoding UTF-8 explícito) garantiza que dos inputs distintos jamás compartan clave. La forma idiomática moderna es `HexFormat.of().formatHex(MessageDigest.digest(bytes))` desde Java 17.

**Atomicidad SET+TTL**:
> En Redis, siempre que exista la variante atómica del comando, úsala en vez de dos comandos separados. Concretamente, `SET key value EX seconds` en una llamada evita la ventana donde el proceso podría morir entre un `SET` y un `EXPIRE` posterior, dejando la clave sin TTL como leak permanente. En Spring Data Redis, `opsForValue().set(key, value, ttl)` mapea directamente a ese comando atómico.

**Fail-safe cache degradation**:
> Si la caché no es fuente de verdad, ninguna petición debe fallar por caída de la caché. Preferir respuesta lenta a respuesta fallida. Implementación: try/catch alrededor de cada operación Redis, log WARN con clave y stack trace, tratar como miss en get y como noop en put, seguir al backend. Se distingue en dashboards: WARN del logger del componente de caché = degradación (aceptable); ERROR = petición rota (alertar). El anti-patrón a evitar es log-and-throw, que produce logs duplicados en capas superiores.

**Spring Data Redis operations grouping**:
> Spring Data Redis expone operaciones agrupadas por tipo de dato Redis a través de `opsForValue()`, `opsForList()`, `opsForHash()`, `opsForSet()`, `opsForZSet()`. Cada uno devuelve una interfaz especializada (`ValueOperations`, `ListOperations`...) con solo los comandos aplicables a esa estructura, mapeados internamente a los comandos Redis subyacentes. Es la forma idiomática de trabajar en vez de comandos planos.

**Lettuce reconexión automática**:
> Lettuce, el cliente Redis por defecto de Spring Data Redis desde Spring Boot 2.0, tiene un `ConnectionWatchdog` que detecta desconexiones y reintenta reconectar automáticamente. En producción esto es clave: reinicios de Redis y failovers son transparentes para la aplicación siempre que Lettuce pueda reconectar. Verificable empíricamente: `docker compose stop redis` → WARN de infra en logs → `docker compose start redis` → reconexión sin reiniciar la app.

## Estado de la app y roadmap

- `document-analyzer-ai` ahora con cache-aside en `POST /analyze`. Impacto medible: cache hit ~5ms vs miss ~3s (aceleración ~500x, y ahorro de coste de tokens Anthropic API por hit).
- **Bloque 4 Redis: 3/3 subtemas cerrados** (RDB/AOF, replicación master-réplica, cache-aside). Bloque cerrado en su forma canónica.
- Sentinel y Cluster cubiertos conceptualmente en la parte 2 pero no en lab (out of scope acordado).
- Cache stampede mencionado conceptualmente en esta parte pero no implementado (out of scope acordado).
- `/analyze/pdf` con cache-aside: pendiente que Tole lo haga solo aplicando el mismo patrón + hashing sobre bytes.
- Roadmap post-Git/Bash oficial (AWS → Kubernetes → OAuth2 → Observability) no tocado hoy. AWS Sesión 11 (Terraform remote state con S3+DynamoDB) sigue siendo el siguiente hito natural fuera del arco Redis.

## Correcciones y aprendizajes de proceso

- **Feedback recibido a media sesión**: "prefiero ir más lento y no que me pongas el código directamente". Aceptado y aplicado: a partir de ese punto, Claude dejó de pegar cuerpos de métodos completos y volvió a Sócrates puro (guía + estructura vacía + Tole implementa). Aprendizaje: pegar código "para acelerar" en un contexto pedagógico Sócrates es contraproducente, aunque la intención sea ayudar.
- **Confusión terminológica sobre "componente" y "get/put"**: Tole preguntó "¿qué componente?" al no reconocer el término aplicado a `CvAnalysisCache`. Culpa de Claude por usar "componente" sin haberlo introducido explícitamente en el contexto. Aclarado. Segundo malentendido similar: "¿los get/put no los toco porque ya están en Spring Data?" — confundió `ValueOperations.get()` (Spring) con `CvAnalysisCache.get()` (propio). Aclarado con la analogía de capas: los métodos propios envuelven los de Spring.
- **Confusión de tiempos en la primera batería de verificación**: los tiempos que Tole pegó al principio no coincidían con lo esperado (primer curl reportaba 0.009s siendo "primer miss"). Diagnóstico correcto vía timestamps de MONITOR: no era realmente miss, era hit disfrazado por orden de ejecución. Regla aprendida: `FLUSHALL` antes de cualquier lab de verificación de caché, y ejecutar curls en orden estricto anotando tiempos. Aplicado en la segunda batería, sin confusiones.
- **Vocabulario**: Claude usó "loguo" (incorrecto). Corrección: "logueo" (castellanización habitual de "to log"), aceptado como jerga técnica; en registro formal se prefiere "registro en el log" / "registrar en el log". Anotado para no reincidir.
- **Fatiga emocional detectada al arrancar** ("Candidaturas... mejor no hablar"). Claude respetó la señal sin insistir y se ofreció a hablar del tema si quería. Tole optó por seguir con lo técnico. Buena calibración de la sesión: 3h de trabajo denso sin abrir la caja del job search hasta que Tole decidiera.

## Próxima sesión

Bloque 4 Redis oficialmente cerrado (3/3 subtemas). Opciones ordenadas por continuidad:

1. **`/analyze/pdf` con cache-aside** — follow-up autónomo, Tole lo hará solo aplicando el patrón + hashing sobre bytes. Duración estimada 60-90 min. Buen ejercicio de consolidación.
2. **AWS Sesión 11**: Terraform remote state con S3 + DynamoDB. Continuación del roadmap post-Git/Bash oficial. Hands-on, cabe en 2h.
3. **Investigar bug `LlmLoggingAdvisor` × `MessageChatMemoryAdvisor`** en `/chat` — con `MONITOR` y `getOrder()`. 45-60 min. Cerraría deuda técnica antigua del arco Redis.
4. **Mini-sesión de reactivación** de algún tema oxidado (JWT refresh, JPA, Testing, Resilience4j, MapStruct, MongoDB, Docker, GitHub Actions, MDC, Actuator, springdoc, Bucket4j). Q&A socrático con código delante. Formato ideal para energía baja / afternoon session.

Ver también `PromptContinuacion-*.md` cuando se genere el de la próxima sesión (no creado aún — pendiente de decisión sobre qué priorizar).
