# Sesión Puente Redis (part-2) — Laboratorio empírico con redis-cli

**Fecha**: 2026-08-10
**Duración**: ~2h30min (arranque ~07:35, cierre ~10:00)
**Fase**: sesión puente Redis, bloque 2 de un arco de 3-4 sesiones
**Bloque**: Redis en Docker + redis-cli (laboratorio)
**Cobertura de la sesión**: laboratorio empírico con Redis 7.4 corriendo en Docker Compose. Ejercicios sobre los cinco tipos de datos verificando comportamiento en vivo desde `redis-cli`. Ejercicio de consolidación final resolviendo un rate limiter fixed window completo. Descubrimiento del principio general "Redis proporciona primitivas, la lógica vive en la app". **Pendiente para bloque 3**: swap real del `ChatMemory` en `document-analyzer-ai` con Spring Data Redis + Lettuce (cierre de la deuda pedagógica de Fase 1). Antes del bloque 3, subsesión aparte para aterrizar en Java la sintaxis del pseudo-código que apareció al final de esta sesión.

Sin commits en esta sesión — laboratorio transversal, no artefacto de portfolio.

---

## Contexto de arranque

Al arrancar (~07:35):

- Diario del bloque 1 leído y revisado por Tole.
- Cabeza fresca, había dormido bien, día libre completo por delante.
- Actualización sobre María Torres Godino (Hays Madrid): sí se envió mensaje al aceptar la conexión el 08/08 (memoria previa incorrecta indicaba lo contrario). Mensaje pasivo, tipo "estaré atento a las ofertas de empleo, a ver si encajo en alguna". Anotado: si hay segunda ronda de contacto conviene ser más específico sobre sector (banca/consultoría), stack (Java backend, Spring Boot, microservicios, AWS) y ubicación (remoto desde España).
- Contrato del arco de aprendizaje intacto: aprender Redis, la deuda de `document-analyzer-ai` es anclaje, no fin.

---

## Setup del laboratorio

### Ubicación

Directorio `~/proyectos/redis-lab/`, **fuera del hub del AI Engineer roadmap y fuera de `cloud-roadmap`**. Es aprendizaje transversal, no artefacto de portfolio. Se destruirá cuando el arco se cierre.

### `docker-compose.yml` minimalista

```yaml
services:
  redis:
    image: redis:7.4-alpine
    container_name: redis-lab
    ports:
      - "6379:6379"
```

Cuatro decisiones lockeadas:

1. **`redis:7.4-alpine`**: versión estable actual, variante Alpine (~15MB en lugar de ~120MB de la variante Debian). Alpine es idiomático en producción también, salvo dependencia inusual del sistema base.
2. **`container_name: redis-lab`**: nombre explícito, no depende del auto-generado por compose. Cómodo para `docker exec`.
3. **`ports: "6379:6379"`**: puerto canónico de Redis. Como 5432 Postgres o 3306 MySQL — memorización obligatoria para entrevistas.
4. **Sin volume**: intencional. Refuerza empíricamente el punto conceptual "persistencia es opt-in en Redis". Al bajar el contenedor con `docker compose down`, todos los datos se pierden.

### Regla operacional recuperada

Al arrancar por primera vez con Docker Desktop apagado, `docker compose up -d` falló con:

```
The command 'docker' could not be found in this WSL 2 distro.
```

WSL2 no tiene Docker instalado nativamente. La integración la aporta Docker Desktop en Windows. Cuando Desktop está apagado, el comando `docker` no existe en WSL. **Primer check operacional cuando `docker` no responde en WSL: verificar Docker Desktop en la barra de tareas de Windows.**

### Ping de sanidad

```
docker compose exec redis redis-cli PING
→ PONG
```

`PONG` = Redis vivo y aceptando comandos. Convención con nombre reconocible universalmente (misma respuesta que un ping de red).

Entrada al modo interactivo:

```
docker compose exec redis redis-cli
127.0.0.1:6379>
```

---

## Ejercicio 1 — Strings + INCR + TTL

### La muerte de una clave en vivo

Rate limiter fixed window en `ratelimit:user:123` con TTL de 60s. Tras 17 `INCR` sucesivos (contador subiendo 1→17) y más de 60 segundos desde el `SET` inicial, un `GET` devolvió `(nil)`. **Redis borró la clave sola**, sin cron, sin código de limpieza, sin intervención. El punto conceptual "los datos efímeros no se resetean, se dejan morir por TTL" observado empíricamente.

### El bug del reset accidental

Al ejecutar el `SET` inicial se omitió el flag `NX` por descuido. Como la clave no existía, el resultado fue el mismo. Se usó como oportunidad pedagógica para verificar qué pasa cuando el `SET` se re-ejecuta sobre una clave viva.

Secuencia crítica:

```
SET ratelimit:user:456 1 EX 60      ← inicializa (sin NX)
INCR ×4                              ← contador = 5
SET ratelimit:user:456 1 EX 60      ← re-inicializa: sobrescribe el 5 y vuelve a 1
GET ratelimit:user:456              ← devuelve "1"
```

**En un rate limiter real, si el `SET` se ejecuta sin `NX` en cada petición del usuario, el contador se resetea a 1 en cada llamada**. El límite nunca se alcanza. El rate limiter deja de existir aunque el código parezca correcto.

Confirmación del `NX`:

```
DEL ratelimit:user:456                    ← borra clave (idempotente)
SET ratelimit:user:456 1 EX 60 NX        ← OK (no existía)
SET ratelimit:user:456 1 EX 60 NX        ← (nil) (ya existía, no toca nada)
```

`(nil)` como respuesta a `SET NX` no es error: es la señal de que la condición no se cumplió. La app puede usarla para distinguir "era la primera vez" (`OK`) de "ya había una clave" (`(nil)`).

### Los tres valores especiales de `TTL`

Observados en vivo:

- **Número positivo** (25, 12, 7, 3, 0...): segundos restantes hasta expiración.
- **`-1`**: la clave existe pero no tiene TTL configurado. Vive para siempre.
- **`-2`**: la clave no existe (nunca existió o expiró y fue borrada).

Confundir `-1` con `-2` es un bug clásico en producción: "creía que la clave estaba viva sin TTL pero en realidad no existía".

Comando complementario `PERSIST clave`: quita el TTL de una clave sin borrarla. Devuelve `1` si sí había TTL para quitar, `0` si no había nada que quitar (idempotencia informativa).

### Patrón general emergente: el "vacío" según el tipo de retorno

Con la clave ya muerta, tres comandos devuelven cosas distintas:

- `HGET clave campo` → `(nil)` (ausencia de valor)
- `HGETALL clave` → `(empty array)` (colección vacía)
- `EXISTS clave` → `(integer) 0` (contador con valor cero)

Regla mental locked: **en Redis, la respuesta a "no hay nada" depende del tipo de retorno del comando, no es una constante universal**. `(nil)` para valores individuales, `(empty array)` o `(empty list)` para colecciones, `0` para contadores. Distinción sutil que aparece en preguntas de entrevista tipo "¿qué devuelve `EXISTS` cuando la clave no existe?".

---

## Ejercicio 2 — Hash

### `HSET` idempotente y aditivo

Comportamiento verificado:

```
HSET user:tole username "Tole" email "..." city "Reinosa" age 48
→ (integer) 4   ← cuatro campos nuevos creados

HSET user:tole username "Tole" email "..." city "Reinosa" age 48   ← mismo comando
→ (integer) 0   ← cero campos nuevos, todos ya existían

HSET user:tole email "nuevo@..." phone "+34600000000"
→ (integer) 1   ← un campo nuevo (phone), el otro (email) ya existía
```

**El número que devuelve `HSET` es cuántos campos son estrictamente nuevos, no éxito o fracaso**. Un `HSET` que actualice 100 campos existentes devuelve `0` y aun así ha sido un update masivo perfectamente exitoso. Consecuencia: no usar el valor de retorno de `HSET` para saber si "la operación tuvo efecto".

Cinco formas de leer un hash, cada una con su propósito:

- `HGET clave campo` → un campo concreto.
- `HMGET clave campo1 campo2 ...` → varios campos específicos.
- `HGETALL clave` → todos los campos y valores.
- `HKEYS clave` → solo nombres de campos.
- `HVALS clave` → solo valores.

### `HGETALL` como anti-patrón en producción

Redis es **single-threaded**: todos los comandos se ejecutan uno detrás de otro en un solo hilo. Mientras Redis procesa un `HGETALL` de un hash gigante (100.000 campos), **ningún otro cliente puede ejecutar nada**. Todos esperan.

Consecuencia práctica: `HGETALL`, `KEYS *`, `SMEMBERS` de sets grandes y `LRANGE 0 -1` de listas grandes son **anti-patrón en producción sobre estructuras de tamaño desconocido o ilimitado**.

Alternativas idiomáticas:

- Si sabes qué campos quieres: `HGET` o `HMGET`. Coste O(1) o O(N sobre campos pedidos), no sobre tamaño del hash.
- Si necesitas iterar sin bloquear: `HSCAN`, que devuelve trozos pequeños. Análogos: `SSCAN`, `ZSCAN`, `SCAN` para el keyspace completo.

**Regla mental refinada**: no es "nunca uses `HGETALL`", es **"conoce el tamaño típico y el máximo posible de la estructura antes de hacer `HGETALL`"**. Sobre un hash pequeño y acotado (5-10 campos de un usuario, series a medias de un cliente) es perfectamente seguro. Sobre estructuras que pueden crecer sin límite es una bomba de tiempo.

### TTL sobre hash: todo o nada

Verificado empíricamente: `EXPIRE user:tole 30` + espera → todos los campos desaparecen simultáneamente. **El hash es una unidad atómica de vida**. No hay "algunos campos sobreviven al vencimiento".

Consecuencia de diseño (retomando el ejercicio del streaming de ayer): si "un Hash por usuario, series como campos" (Opción B), todas las series comparten TTL. No se puede tener "Breaking Bad renovada hace 10 días" y "The Wire renovada hace 80 días" coexistiendo en el mismo hash. Es todo o nada. Redis 7.4 introduce `HEXPIRE` para expirar campos individuales, pero es reciente y no siempre disponible.

---

## Ejercicio 3 — List (el patrón `ChatMemory`)

### `LPUSH` inserta por la izquierda

Cuatro `LPUSH` sobre `conversation:sesion-1` en orden temporal. Resultado tras `LRANGE 0 -1`:

```
1) "bot: una base de datos en memoria"    ← posición 0 (último insertado)
2) "usuario: qué es Redis"                 ← posición 1
3) "bot: hola, cómo puedo ayudarte"        ← posición 2
4) "usuario: hola"                         ← posición 3 (primero insertado)
```

**`LPUSH` = Left PUSH**: inserta por la cabeza. El elemento nuevo queda en posición 0. Los viejos se desplazan a posiciones más altas. **Consecuencia: `LRANGE 0 N` devuelve los N+1 más recientes en orden decreciente (nuevo primero)**.

Contraparte simétrica: `RPUSH` = Right PUSH, inserta por la cola. Y sus operaciones asociadas: `LPOP`/`RPOP` para sacar por cada extremo.

### La elección `LPUSH` vs `RPUSH` no es estilística

Dos patrones idiomáticos que dependen de la elección:

- **`LPUSH` + `LRANGE 0 N`** implementa "los N más recientes". Timelines, feeds, notificaciones, chat memory.
- **`RPUSH` + `LPOP`** implementa cola FIFO. Colas de trabajo, procesamiento de tareas.

**Confundirlas invierte el orden lógico de lectura**.

### Patrón `capped list` observado en vivo

```
LTRIM conversation:sesion-1 0 2
→ OK
LRANGE conversation:sesion-1 0 -1
→ (3 elementos, los tres más recientes)
LLEN conversation:sesion-1
→ (integer) 3
```

`LTRIM` no borra por elementos, **borra por rango de posiciones**. Le dices "quédate con estas posiciones, tira el resto". Índices coherentes con `LRANGE`: 0 es la izquierda, -1 es la última.

**Patrón `capped list` completo**:

```
LPUSH conversation:{id} "<mensaje nuevo>"
LTRIM conversation:{id} 0 9              ← mantener solo los 10 más recientes
```

Estas dos operaciones ejecutadas en cada mensaje nuevo. Resultado: la lista **nunca crece más de 10 elementos**. Se autolimita. Descarta el más antiguo cuando entra uno nuevo. **Es la estructura base del `ChatMemory` que se implementará en el bloque 3.**

### Introducción a `MULTI`/`EXEC`

Para atomicidad del par `LPUSH` + `LTRIM` en producción (evitar que otro cliente meta comandos entre medias y rompa la invariante temporal), Redis ofrece transacciones:

```
MULTI                                       ← abre transacción
LPUSH conversation:{id} "..."               → QUEUED (no ejecuta, cola)
LTRIM conversation:{id} 0 9                 → QUEUED
EXEC                                        ← ejecuta todo atómicamente
```

**Matiz importante**: las transacciones Redis **no son ACID como Postgres**.

- **Aislamiento sí**: ningún comando externo se ejecuta entre `MULTI` y `EXEC`.
- **Atomicidad parcial**: si un comando falla en runtime, los demás **se ejecutan igualmente**. No hay rollback.

Para lógica compleja con validaciones condicionales, alternativa idiomática: **scripts Lua con `EVAL`**, que sí ejecutan atómicamente y permiten condicionales. Detalle avanzado no explorado en esta sesión.

---

## Ejercicio 4 — Set

### Comandos básicos

```
SADD article:42:tags "redis" "cache" "backend" "spring"
→ (integer) 4                      ← cuatro elementos nuevos
SADD article:42:tags "redis"
→ (integer) 0                      ← duplicado ignorado silenciosamente
```

Consistencia con `HSET`: `SADD` devuelve **cuántos elementos son estrictamente nuevos**, no éxito.

- `SMEMBERS clave` → elementos del set (colección).
- `SCARD clave` → tamaño (contador). Viene de **cardinalidad**, término matemático para "número de elementos de un conjunto".
- `SISMEMBER clave elemento` → `1` si está, `0` si no. Se lee "**S IS MEMBER**" — pregunta binaria.

### Operaciones de conjuntos idiomáticas

```
SINTER article:42:tags article:99:tags   ← intersección: elementos en AMBOS
SUNION article:42:tags article:99:tags   ← unión: elementos en CUALQUIERA
SDIFF article:42:tags article:99:tags    ← diferencia: en A pero NO en B
```

**Importante — devuelven los elementos, no un contador**. Si se necesita solo el tamaño del resultado: `SINTERSTORE` en clave temporal + `SCARD`, o hacer `SINTER` + contar en la app.

**`SDIFF` es asimétrico**: `A \ B ≠ B \ A`. Redis respeta la semántica matemática. `SINTER` y `SUNION` son simétricos.

### El poder real de los sets

Casos que en Postgres requieren un JOIN con INTERSECT/UNION/EXCEPT, en Redis son un solo comando en microsegundos:

- Red social: "amigos comunes entre Alice y Bob" → `SINTER friends:alice friends:bob`.
- Banca (venta cruzada): "clientes con producto A y producto B" → `SINTER holders:productA holders:productB`.
- Segmentación negativa: "usuarios premium pero no en lista de morosos" → `SDIFF users:premium users:morosos`.

Rendimiento: prácticamente constante hasta sets con millones de elementos. `SINTER` optimizado internamente para empezar por el set más pequeño.

### Redis no valida claves inexistentes

Diagnóstico en vivo: la primera vez que `SINTER article:42:tags article:99:tags` devolvió `(empty array)`, la causa no fue reinicio del contenedor ni `FLUSHDB` — fue un typo latente. Existían `articulo:42:tags` (español, primer intento) y `article:99:tags`, pero no `article:42:tags`. `SINTER` sobre un set inexistente devuelve intersección con conjunto vacío = vacío.

**Regla mental locked**: **Redis trata las claves inexistentes como estructuras vacías del tipo esperado por el comando**. No lanza error. Ventaja: código más simple. Desventaja: **un typo en un nombre de clave se manifiesta como resultado vacío, no como fallo**. Contraste con Postgres, que devolvería `ERROR: relation "artculo" does not exist`. Redis no tiene esa red de seguridad.

---

## Ejercicio 5 — Sorted Set

### Orden no es inserción, es puntuación

Corrección importante durante la sesión: al predecir el output de `ZRANGE`, se asumió inicialmente "orden de inserción". Falso.

```
ZADD leaderboard 1500 "tole"
ZADD leaderboard 2200 "carlos"
ZADD leaderboard 1800 "maria"
ZADD leaderboard 900 "juan"
ZADD leaderboard 2500 "ana"

ZRANGE leaderboard 0 -1 WITHSCORES
→ juan 900, tole 1500, maria 1800, carlos 2200, ana 2500
```

Ana fue el último insertado, pero aparece la última en `ZRANGE` porque tiene la puntuación más alta y `ZRANGE` ordena por puntuación **ascendente**. `ZREVRANGE` invierte el orden (descendente, mayor primero).

**Regla mental locked**: en Sorted Set, **el orden es propiedad emergente de las puntuaciones**, no una lista con historia de inserción. Redis mantiene la estructura ordenada internamente (implementación con skip list + hash table). Insertar es O(log N), actualizar puntuación reubica en O(log N).

### Familia completa de comandos ZSet

- `ZRANGE clave start stop` → rango por posición, orden ascendente.
- `ZREVRANGE clave start stop` → rango por posición, orden descendente.
- `WITHSCORES` → incluye la puntuación en el output.
- `ZRANK clave miembro` → posición del miembro en orden ascendente.
- `ZREVRANK clave miembro` → posición en orden descendente (el "puesto" del leaderboard humano).
- `ZSCORE clave miembro` → puntuación literal.
- `ZRANGEBYSCORE clave min max` → rango por puntuación (no por posición).

### Casos de uso más allá del leaderboard obvio

Emergen constantemente en arquitectura:

- **Cola de prioridad**: la puntuación es la urgencia. `ZREVRANGE 0 0` saca el trabajo más urgente.
- **Timeline por timestamp**: la puntuación es un Unix timestamp. `ZRANGEBYSCORE clave t1 t2` devuelve eventos en una ventana temporal.

**Analogía Java**: cualquier problema resoluble con `TreeMap<Double, T>` o `PriorityQueue<T>` es candidato natural a Sorted Set de Redis, con la ventaja adicional de ser persistente y compartido entre instancias.

---

## Ejercicio final — Rate limiter completo

Consolidación sin ayuda. Tarea: implementar fixed window rate limiter con 60s de ventana y 5 llamadas/minuto para usuario `999`.

Resolución completa en `redis-cli`:

```
SET user:999 1 EX 60 NX     → OK
INCR user:999 (×5)          → 2, 3, 4, 5, 6
TTL user:999                → 41 (segundos restantes)
SET user:999 1 EX 60 NX     → (nil)   ← intento de re-inicializar mientras clave viva: NO hace nada, correcto
[espera > 60s]
TTL user:999                → -2       ← clave muerta
SET user:999 1 EX 60 NX     → OK       ← nueva ventana arrancada
GET user:999                → "1"
```

### La pregunta clave que reveló el principio general

Al terminar: "¿pero cómo limito a 5?". La resolución en Redis estaba completa; el "límite" no vive ahí.

**Descubrimiento central**: **Redis no limita a 5. Redis solo cuenta. Es la aplicación la que lee el contador devuelto por `INCR` y decide qué hacer**.

Ejemplo mostrado en pseudo-código Spring Boot (aterrizaje real en Java aplazado a subsesión aparte):

```java
Long count = redis.opsForValue().increment(key);
if (count > 5) return ResponseEntity.status(429).body("Too many requests");
```

Cambiar el umbral de 5 a 10 es un cambio en Java, no en Redis.

**Principio general**: **Redis proporciona primitivas eficientes (`INCR`, `ZADD`, `SADD`, TTL, `SINTER`). La lógica de negocio vive en la aplicación**. Contraste con Postgres, donde constraints, triggers y funciones almacenadas permiten empotrar lógica en la base de datos.

---

## Cierre del laboratorio

```
FLUSHDB                    ← borra todo el keyspace (peligroso en prod, útil en lab)
KEYS *                     ← (empty array)
exit                       ← salida del redis-cli
docker compose down        ← para y elimina el contenedor
```

Sin volume: todos los datos perdidos, como se pretendía. La imagen queda cacheada; próximo `up` es instantáneo.

`FLUSHDB` en producción está típicamente deshabilitado o requiere autenticación especial. Comando destructivo consciente.

---

## Reglas operacionales lockeadas nuevas

### Docker Desktop es dependencia implícita de WSL para `docker`

Cuando `docker` no responde en WSL, primer check operacional es Docker Desktop en la barra de tareas de Windows. La integración WSL2 ↔ Docker vive en Desktop.

### `SET NX` no es opcional en rate limiter

Sin `NX`, cada `SET` en el flujo del usuario **sobrescribe el contador y lo devuelve a 1**. El límite nunca se alcanza. El rate limiter deja de existir sin que salte error alguno. El `NX` es la única barrera contra este bug.

### `HGETALL`, `KEYS *`, `LRANGE 0 -1`, `SMEMBERS` son anti-patrón en producción sobre estructuras grandes

Redis es single-threaded. Comandos que devuelven colecciones enteras bloquean a todos los demás clientes durante su ejecución. Alternativas idiomáticas: `HSCAN`, `SCAN`, `SSCAN`, `ZSCAN` iteran en trozos sin bloquear.

### El "vacío" en Redis depende del tipo de retorno

`(nil)` para valores individuales, `(empty array)` para colecciones, `0` para contadores. No hay constante universal de "vacío".

### Redis no valida claves inexistentes

Operar sobre una clave inexistente devuelve estructura vacía del tipo esperado, sin error. **Un typo en un nombre de clave se manifiesta como resultado vacío, no como fallo**. Debugging: revisar nombres cuando resultados vacíos aparezcan sin explicación.

### `LPUSH` vs `RPUSH` no es estilístico

Define el patrón funcional de la lista. `LPUSH` + `LRANGE 0 N` = "últimos N recientes" (chat memory, feeds, timelines). `RPUSH` + `LPOP` = cola FIFO (colas de trabajo).

### Orden en Sorted Set es propiedad emergente de puntuaciones

No hay orden de inserción. Actualizar una puntuación reubica el elemento automáticamente en O(log N). Confundir "orden de inserción" con "orden por puntuación" es error frecuente al aprender.

### Redis proporciona primitivas, la lógica vive en la app

Umbrales, reglas, interpretación del significado de contadores/scores, manejo de errores → todo en la app. Redis se mantiene simple, rápido, agnóstico al dominio. Contraste con Postgres, donde la BD puede empotrar lógica.

---

## Frases ⭐⭐⭐ para entrevistas

- **Redis single-threaded y comandos peligrosos**: "Redis es single-threaded: un comando lento bloquea todos los demás clientes. Comandos que devuelven colecciones enteras — `HGETALL`, `KEYS *`, `LRANGE 0 -1`, `SMEMBERS` — son anti-patrón en producción sobre estructuras grandes. Alternativa idiomática: `HSCAN`/`SSCAN`/`ZSCAN` iteran en trozos sin bloquear el servidor".
- **Patrón capped list para chat memory**: "El patrón `capped list` en Redis se implementa con `LPUSH clave valor` + `LTRIM clave 0 N-1` en cada inserción. La lista se autolimita a N elementos, descartando el más antiguo cuando entra uno nuevo. Es el patrón canónico para `ChatMemory`, timelines con límite, feeds de notificaciones y logs rotatorios. En producción, ambos comandos en `MULTI`/`EXEC` para atomicidad".
- **Transacciones Redis no son ACID**: "Las transacciones en Redis (`MULTI`/`EXEC`) garantizan aislamiento — ningún comando externo se cuela entre medias — pero no atomicidad ACID: si un comando falla en runtime, los otros se ejecutan igualmente, sin rollback. Para lógica compleja con validaciones, la alternativa idiomática son scripts Lua con `EVAL`, que sí ejecutan atómicamente y permiten condicionales".
- **Sets sustituyen JOINs de teoría de conjuntos**: "Los sets de Redis con `SINTER`, `SUNION`, `SDIFF` resuelven queries de teoría de conjuntos en microsegundos, comportamiento prácticamente constante hasta millones de elementos. Sustituyen operaciones que en Postgres requieren JOINs con INTERSECT/UNION/EXCEPT. Casos típicos: amigos comunes en redes sociales, venta cruzada (clientes con producto A y B), segmentación negativa (usuarios en segmento X pero no en lista Y). `SDIFF` es asimétrico; `SINTER` y `SUNION` son simétricos".
- **Redis no valida claves**: "Redis no valida existencia de claves: operar sobre una clave inexistente devuelve una estructura vacía del tipo esperado, sin error. Ventaja: código más simple. Desventaja: un typo en un nombre de clave se manifiesta como resultado vacío, no como fallo. Depurar `SINTER article:42 article:99` que devuelve `(empty array)` requiere revisar los nombres — Redis nunca dirá que la clave no existe. Contraste con Postgres, que lanza `ERROR: relation does not exist`".
- **Sorted Sets: orden como propiedad emergente**: "Los Sorted Sets de Redis no tienen orden de inserción: el orden es propiedad emergente de las puntuaciones asociadas. Redis mantiene la estructura ordenada internamente (skip list + hash table). Insertar es O(log N), actualizar puntuación reubica en O(log N). Cualquier ranking, leaderboard, cola de prioridad o timeline por relevancia es un Sorted Set — no hace falta ordenar en la aplicación".
- **Redis primitivas, app lógica**: "Redis proporciona primitivas eficientes; la lógica de negocio vive en la aplicación. `INCR` cuenta pero no rechaza — es la app quien lee el contador y decide si aceptar o devolver 429. Cambiar el límite de un rate limiter de 5 a 10 llamadas por minuto es un cambio en el código Java, no en Redis. Este diseño mantiene Redis simple, rápido y agnóstico al dominio. Contrasta con Postgres, donde constraints, triggers y funciones almacenadas permiten empotrar lógica de negocio dentro de la base de datos".
- **Tres valores especiales del `TTL`**: "El código de retorno de `TTL` en Redis distingue tres estados fáciles de confundir: número positivo (clave viva con TTL definido), `-1` (clave viva sin TTL, persistente), `-2` (clave no existe). En una aplicación real, confundir `-1` con `-2` es un bug clásico: `creía que la clave estaba viva sin TTL, pero en realidad no existía`".
- **`SET NX` como salvaguarda de rate limiter**: "En un rate limiter fixed window, el flag `NX` de `SET` no es opcional: sin él, cada `SET` sobrescribe el contador y lo devuelve a 1, el límite nunca se alcanza y el rate limiter deja de existir sin que salte error alguno. `NX` garantiza que la inicialización ocurra una sola vez por ventana; los `INCR` siguientes trabajan sobre la clave existente".
- **`LPUSH` vs `RPUSH` define el patrón**: "En Redis, `LPUSH` y `RPUSH` no son estilísticos: definen el patrón funcional de la lista. `LPUSH` + `LRANGE 0 N` implementa `los N más recientes` (timelines, chat memory, feeds). `RPUSH` + `LPOP` implementa cola FIFO (colas de trabajo, procesamiento de tareas). Confundirlas invierte el orden lógico de lectura".

---

## Próxima subsesión

**Subsesión intermedia — Aterrizaje en Java del pseudo-código del rate limiter.**

Objetivo: bajar la lógica de "Redis cuenta, app decide" a sintaxis real de Spring Boot + Spring Data Redis. Configuración del cliente Lettuce, inyección de `RedisTemplate` o `StringRedisTemplate`, operaciones equivalentes a los comandos vistos en el laboratorio, manejo de errores. Sin implementar todavía el `ChatMemory` — solo aterrizar las primitivas.

Densidad estimada: media. Sesión corta a media.

**Bloque 3 (después) — Swap real del `ChatMemory` en `document-analyzer-ai` con Spring Data Redis.**

- Añadir servicio `redis` al `docker-compose.yml` de `document-analyzer-ai`.
- Dependencia `spring-boot-starter-data-redis` con Lettuce (default).
- Diseño de claves: `chat:memory:{conversationId}` como Redis List con `LPUSH` + `LTRIM 0 9`.
- Serialización de mensajes (JSON típicamente, valorar `Jackson2JsonRedisSerializer` vs texto plano).
- Implementación de un `ChatMemory` personalizado que Spring AI acepte, respaldado por Redis.
- TTL por conversación (24h típicamente para chats interactivos).
- Verificación empírica: mensaje enviado → aparece en Redis → reinicio de app → memoria persiste.

**Cierre de la deuda pedagógica de Fase 1 al terminar el bloque 3**.

**Bloque 4 (opcional)**: RDB/AOF, replicación básica, cache-aside en banca. Solo si energía y contexto lo permiten.

---

**Estado emocional al cierre**: sesión larga (~2h30min) pero de manos, cognitivamente más ligera que la conceptual de ayer. Buenas señales de aprendizaje: predicciones activas antes de cada comando, autocorrección al ver output distinto de lo esperado (orden en `ZRANGE`, "número" vs "elementos" en `SINTER/SUNION/SDIFF`), diagnóstico independiente del typo `articulo` vs `article`. La pregunta final "¿cómo limitar a 5?" abrió el principio general más importante de la sesión — Redis proporciona primitivas, la app decide — y valió por sí sola el ejercicio de consolidación. Aplazamiento explícito de sintaxis Java a subsesión aparte: buena decisión, evita mezclar aprendizaje de Redis con aprendizaje de cliente Java. Contrato del arco intacto: bloque 3 cerrará la deuda de `document-analyzer-ai` usando exactamente el patrón capped list visto hoy.
