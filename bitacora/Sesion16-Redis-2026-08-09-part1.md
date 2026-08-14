# Sesión Puente Redis (part-1) — Mapa mental conceptual

**Fecha**: 2026-08-09
**Duración**: dos tramos con día fuera intercalado — mañana (~08:40, arranque contextual y reformulación de scope) + tarde (~18:30-20:00, ~1h30min de bloque conceptual denso)
**Fase**: sesión puente entre S15 (`erp-mcp-server` cerrado) y S16 (agente ReAct en `erp-purchasing-agent`)
**Bloque**: Redis — sesión 1 de un arco de 3-4 planificadas
**Cobertura de la sesión**: mapa mental completo de Redis sin tocar código. Qué es, tres casos de uso principales, cinco tipos de datos, comportamiento del TTL con ejemplos concretos (rate limiter fixed window, sesión de usuario con renovación, ejercicio de diseño de claves en escenario de streaming). Cero setup, cero Docker, cero `redis-cli`. **Pendiente para sesión 2**: Redis en Docker Compose + jugar con los tipos de datos en vivo desde `redis-cli`. **Sesión 3**: swap real del `ChatMemory` en `document-analyzer-ai` con Spring Data Redis (cierra la deuda pedagógica de Fase 1). **Sesión 4 opcional**: profundidad para CV (persistencia RDB/AOF, replicación básica, cache-aside en banca).

Sin commits en esta sesión — solo aprendizaje conceptual.

---

## Contexto de arranque

Al arrancar el tramo de mañana (~08:40):

- S15-part-5 recién cerrada con commit `ff35c28` del README de `erp-mcp-server`, pusheado a `origin/main`. Bloque S15 completo cerrado tras cinco subsesiones.
- Diario de S15-part-5 escrito y guardado en el hub.
- Descansado, más horas de sueño de lo habitual.
- Domingo libre completo.
- 4 rechazos de candidaturas recibidos el sábado. Interpretados como ruido de ATS con filtrado automático, no señal de mercado.
- Se pasó día fuera durante la parte central. Retomada la ventana a las 18:30.

Estado emocional al arrancar la parte conceptual: cabeza limpia, disposición a aprender desde cero ("no sé ni lo que es").

---

## Reformulación del scope de la sesión puente (mañana)

Antes de arrancar el mapa mental, decisión de scope importante.

El prompt de continuación de S15 declaraba: "Sesión puente Redis en `document-analyzer-ai` — 1-1.5 sesión completa mínima". El asistente arrancó asumiendo que el objetivo era "cerrar la deuda técnica de swap in-memory → Redis" y tocar código en `document-analyzer-ai` desde el principio.

**Corrección explícita del usuario**: el objetivo no es cerrar la deuda; **el objetivo es aprender Redis**. La deuda de `document-analyzer-ai` es el anclaje concreto donde aterrizar los conceptos, no el fin. Si al final la deuda se cierra con tres líneas de config y no se aprende Redis en el proceso, la sesión ha sido un fracaso aunque el código funcione.

Reformulación aceptada. Arco de la sesión puente redefinido en 3-4 bloques con densidad manejable:

1. **Mapa mental conceptual** (esta sesión). Sin código.
2. **Redis en Docker + `redis-cli`**. Laboratorio empírico.
3. **Integración con Spring Boot y swap real del `ChatMemory` en `document-analyzer-ai`**. La deuda se cierra al final de este bloque.
4. **Profundidad para CV** (opcional): persistencia, replicación, cache-aside en banca.

Regla lockeada: **el objetivo dicta el arco, no la deuda**. Cuando el usuario declara "quiero aprender X y de paso saldar Y", saldar Y es consecuencia, no vía.

---

## Tour rápido por `document-analyzer-ai` (mañana)

Refresco del proyecto tras semanas sin tocarlo. Verificación empírica con `find` y `grep`, no confianza en memoria histórica.

Hallazgos:

- **Ocho ficheros Java** en total. Proyecto pequeño.
- **Tres controllers**: `ChatController` (`GET /chat` con `ChatMemory`), `AnalyzeController` (`POST /analyze` sobre texto plano), `AnalyzePdfController` (`POST /analyze/pdf` multimodal). Los dos `/analyze` son fire-and-forget: reciben documento, llaman al LLM, devuelven `CvSummary` en la misma respuesta HTTP. **No persisten nada**.
- **Cero `HashMap` o `ConcurrentHashMap` en el código**. El único `@Component` es `LlmLoggingAdvisor`.
- **La memoria del chat vive dentro del `ChatMemory` de Spring AI**, no en un `Map` que Tole hubiera escrito. Retención implícita: últimos 10 mensajes por `conversationId`.

Corrección importante del entendimiento inicial: la nota "in-memory not Redis" del prompt de continuación se refería **exclusivamente al `ChatMemory` de `/chat`**, no a `/analyze` (que nunca fue diseñado para persistir). La deuda de Redis en este proyecto es específicamente sobre la memoria conversacional.

Regla operacional lockeada: **verificar en el código real antes de asumir arquitectura desde memoria** (propia o del prompt de continuación). Un `grep -rn HashMap src/main/java` cuesta 3 segundos y evita construir explicaciones sobre supuestos que no están en disco.

---

## Bloque conceptual: mapa mental de Redis (tarde)

### 1. Qué es Redis en una frase

**Redis es una base de datos que guarda todo en RAM**. Tres consecuencias inmediatas:

- **Rapidísimo**: microsegundos por operación. Entre 100 y 1000 veces más rápido que Postgres para operaciones simples de lectura/escritura por clave.
- **Capacidad limitada**: todo lo que quieras guardar tiene que caber en la RAM del servidor (menos lo que consume el SO y la propia app).
- **Volátil por defecto**: si el servidor se apaga, la RAM se borra. Redis tiene mecanismos de persistencia opcional (RDB, AOF — se verán en bloque 4), pero la fuente de verdad sigue siendo la RAM.

### 2. Los tres casos de uso principales

Distinción con nombres propios porque emerge en entrevistas continuamente:

**Caso 1 — Caché (cache-aside de una fuente de verdad).** Guardas resultados de operaciones caras (query pesada a Postgres, llamada a API externa, cálculo con CPU). La primera vez pagas el coste, guardas en Redis. Las siguientes vas a Redis primero; si el dato está, lo devuelves en microsegundos. Si desaparece, no pasa nada: regeneras desde la fuente. Redis está "al lado" de Postgres acelerando lo caliente, no reemplazándolo.

**Caso 2 — Datos con TTL natural.** Datos que por su naturaleza tienen fecha de caducidad: códigos de verificación por SMS (5 min), tokens de sesión (24h), contadores de rate limit (1 min), memoria de conversación de un chat mientras el usuario esté activo. Redis borra la clave sola al vencer el TTL — sin cron jobs, sin código de limpieza.

**Caso 3 — Estado compartido entre instancias.** Cuando la app corre en varias réplicas (típico en Kubernetes) y el estado no puede vivir en la RAM de una sola instancia. Redis es el punto central que todas las instancias leen y escriben. Sin sincronización manual entre nodos.

Regla mental locked: **los tres casos no son excluyentes**. Un dato puede caer en varios (la `ChatMemory` cae en TTL natural + estado compartido). Cuantas más razones convergen, más clara la elección de Redis.

### 3. Cuándo Redis NO es la respuesta (caso saldos bancarios)

Ejercicio: "un compañero propone meter los saldos de las cuentas bancarias en Redis para acelerar la consulta desde la app móvil". Analizado con detalle porque es exactamente el tipo de pregunta que se hace en entrevistas de banca.

Tres razones por las que el saldo no puede ser dato principal en Redis:

1. **Fuente de verdad legal y contable**: si Redis dice 1200€ y Postgres dice 1500€, el auditor manda a Postgres. Redis nunca puede ser autoritativo para datos con consecuencias legales.
2. **Volatilidad**: un fallo del cluster no puede significar "hemos perdido los saldos". Impensable en banca.
3. **ACID con múltiples claves**: transferencias atómicas (bajar origen + subir destino) son nativas en Postgres. Redis no ofrece ACID robusto multi-clave en cluster.

**Pero el compañero no está totalmente equivocado.** La respuesta profesional articula el patrón completo:

> "El saldo vive en Postgres. Cacheamos en Redis con TTL corto (ej. 30s) como cache-aside. Consulta va a Redis primero; miss va a Postgres, guarda en Redis, devuelve. Cada escritura en Postgres invalida la clave en Redis. En el peor caso el usuario ve un saldo hasta 30s desactualizado, nunca uno incorrecto persistente."

Estructura del razonamiento a llevarse a entrevista: **fuente de verdad → patrón → TTL como decisión explícita → invalidación como parte del diseño → peor caso caracterizado**.

### 4. Los cinco tipos de datos

Nombres propios que hay que conocer:

- **String**: par clave-valor básico. Soporta también contadores atómicos con `INCR` (base de rate limiters).
- **Hash**: objeto con campos, un `Map<String,String>` dentro de una clave. Útil para representar entidades (usuario, producto).
- **List**: colección ordenada por inserción, con push/pop por ambos extremos. Casos: colas ligeras, "últimos N eventos". **Directamente relevante para `ChatMemory`**: una conversación es una lista ordenada de los últimos N mensajes.
- **Set**: colección sin orden y sin duplicados. Operaciones de intersección/unión en microsegundos (seguidores, tags, "usuarios que vieron X").
- **Sorted Set (ZSet)**: como set pero cada elemento tiene puntuación numérica y están ordenados por ella. Rankings, leaderboards, "top N por prioridad".

**Redis no tiene tablas, JOINs, ni queries SQL**. Si necesitas relacionar cosas, tu app hace dos consultas y las combina en memoria. Redis brilla en accesos por clave conocida; no es la herramienta para queries complejas.

### 5. TTL como diseño activo, no solo caducidad

Punto conceptual central. Dos escenarios con TTL pero comportamientos opuestos:

**Rate limiter (fixed window)**. Contador por usuario que se resetea cada minuto.

```
SET ratelimit:user:123 1 EX 60 NX   ← inicializa ventana solo si no existe
INCR ratelimit:user:123              ← incrementa en llamadas sucesivas
```

Cuando pasan 60 segundos, **Redis borra la clave sola**. La próxima petición del usuario ejecuta `INCR` sobre una clave inexistente → Redis la crea con valor 1 → nueva ventana. No hay cron, no hay "poner el contador a cero", no hay job de reseteo. **La clave muere y renace.**

**Sesión de usuario con renovación**. TTL de 24h que se empuja en cada petición del usuario.

```
# Al login:
SET session:user:123 "<json>" EX 86400 NX

# En cada petición autenticada:
EXPIRE session:user:123 86400    ← empuja el TTL 24h más adelante desde AHORA
# O más limpio:
GETEX session:user:123 EX 86400  ← lee valor + empuja TTL, atómico
```

Mientras haya actividad, la clave sobrevive. Si el usuario desaparece 24h sin hacer nada, la clave muere sola y hay que volver a hacer login.

**El mismo mecanismo (TTL) produce dos comportamientos completamente distintos según cuándo se aplica `EXPIRE`**: al crear la clave y no tocar más (rate limiter) vs al crear y renovar en cada acceso (sesión).

### 6. Persistencia por defecto vs caducidad opt-in

Regla mental que sorprende a quien viene de bases de datos tradicionales:

**En Redis, la persistencia es la opción por defecto. La caducidad es opt-in.**

Si guardas una clave sin `EXPIRE` ni flag `EX`, esa clave vive para siempre (hasta llenar RAM o borrado manual). Al revés de lo que uno espera. Consecuencia: si se te olvida poner TTL a una clave que debía ser efímera, se queda ahí llenando la RAM. **Fuga de memoria clásica en aplicaciones Redis mal diseñadas.**

### 7. `SET NX` vs `EXPIRE` vs `GETEX` (matiz importante)

Tres operaciones que se confunden fácil:

- **`SET ... EX ... NX`**: crea una clave con TTL inicial **una sola vez**. Si ya existe, no hace nada (el `NX` = "Not eXists" así lo garantiza). Usado en el login para inicializar la sesión.
- **`EXPIRE`**: empuja el TTL de una clave existente. No crea nada. Si la clave no existe, `EXPIRE` no hace nada (devuelve 0). Usado para renovar sesiones activas.
- **`GETEX ... EX ...`**: lee el valor de la clave y de paso renueva el TTL, en una sola operación atómica. **No crea claves**: si la clave no existe, devuelve `nil` y el `EX` se ignora. Reemplaza al par `GET + EXPIRE` en cada petición autenticada.

Flujo típico de una app con sesiones en Redis:

```
POST /login:
    validar credenciales contra Postgres
    SET session:{id} "<json>" EX 86400 NX
    devolver sessionId al cliente

Cualquier endpoint autenticado:
    resultado = GETEX session:{sessionId} EX 86400
    si resultado == nil → 401 (sesión expirada o inexistente)
    si no → deserializar y continuar
```

**`SET NX` solo en `/login`. `GETEX` en todas las demás peticiones.** Cada comando con su propósito.

Confundir los tres lleva a bugs sutiles:
- Reejecutar `SET` en cada petición **sobrescribe la sesión** cada vez (pierdes datos si los habías modificado).
- Ejecutar `EXPIRE` sobre clave inexistente pasa silenciosamente sin renovar nada.

### 8. Diseño de claves = diseño de schema

Ejercicio final: app de streaming con "guardar en qué minuto quedó el usuario en cada serie". Múltiples series a medias por usuario. TTL de 90 días.

Dos diseños idiomáticos válidos:

**Opción A — Una clave por par (usuario, serie), valor simple.**

```
SET progress:user:123:series:breaking-bad 40 EX 7776000
SET progress:user:123:series:the-wire     12 EX 7776000
```

Ventaja: **TTL independiente por serie**. Si el usuario ve Breaking Bad hoy pero no toca The Wire, solo Breaking Bad se renueva. The Wire caduca a los 90 días desde la última vez que se tocó.

Desventaja: para saber "todas las series a medias del usuario" hay que buscar por patrón con `SCAN` (más costoso).

**Opción B — Un Hash por usuario, campos = series.**

```
HSET progress:user:123 breaking-bad 40 the-wire 12
EXPIRE progress:user:123 7776000
```

Ventaja: `HGETALL progress:user:123` devuelve todas las series en una operación.

Desventaja: **TTL compartido a nivel del hash**. Si el usuario ve una serie, el TTL se renueva para todas.

**En Netflix real probablemente Opción A** por el TTL independiente. Para una app didáctica de estudio, Opción B es más simple.

Regla mental locked: **el diseño de claves en Redis es equivalente al diseño de schema en Postgres**. No es un detalle menor — es donde vive la arquitectura de la solución. La pregunta clave en Redis es "¿qué información va en la clave y qué información va en el valor?", análoga a "¿qué tablas y qué relaciones?" en Postgres.

Correcciones menores sobre sintaxis:
- **`HSET` no acepta `EX`**. El flag `EX` es del comando `SET` (para strings). Para expirar hashes hay que hacer `EXPIRE` en un segundo comando. Redis 7.4 introdujo `HEXPIRE` para expirar campos individuales, pero es reciente y no siempre disponible.
- **No duplicar información identificativa entre clave y valor**. Si `userId` está en la clave `session:user:123`, no lo pongas también como campo del hash. Redundancia inútil y fuente de inconsistencias.

---

## Reglas operacionales nuevas lockeadas

### Objetivo dicta el arco, no la deuda

Cuando el usuario declara "quiero aprender X y de paso saldar Y", **saldar Y es consecuencia**, no vía. El aprendizaje se estructura para maximizar comprensión de X; la deuda se cierra cuando naturalmente encaje en el arco, no al principio como setup.

Aplicación práctica: la sesión puente Redis se estructura como bloque conceptual → laboratorio → integración real (deuda cerrada aquí) → profundidad opcional. No como "swap primero, aprender después".

### Verificar código real antes de asumir desde memoria

Antes de proponer arquitectura o correcciones sobre un proyecto, ejecutar `find` + `grep` sobre `src/main/java` para verificar qué hay realmente. La memoria del prompt de continuación puede tener matices imprecisos: "in-memory not Redis" puede referirse a algo distinto de lo que uno asume. 3 segundos de verificación empírica valen mucho.

### Errores del asistente reconocidos durante la sesión

Anotados por transparencia (útil para calibrar la confianza en futuras sesiones):

- **Orden de proyectos en el roadmap**: se dijo que `document-analyzer-ai` era anterior a task-manager-api y microservicios. Falso: es Phase 1 del AI Engineer roadmap y va **después** en el tiempo. Corregido por el usuario.
- **Placeholder `<datos_sesion>` opaco**: se usó sin explicar y generó confusión sobre si eran credenciales. Se clarificó: son datos derivados del login (userId, username, roles), nunca contraseñas.
- **Ambigüedad en `GETEX` sobre claves inexistentes**: la primera explicación mezcló "renueva TTL" con "crea si no existe". Corregido: `GETEX` no crea claves, devuelve `nil` si la clave no está.

Regla derivada: **cuando el usuario pregunta "no lo entiendo" o "explícalo otra vez", la explicación anterior tenía ambigüedad real, no falta de comprensión del usuario**. Reformular sin defenderse.

---

## Frases ⭐⭐⭐ para entrevistas

- **Redis no es fuente de verdad de datos legales/contables**: "Redis nunca es la fuente de verdad de datos con implicaciones legales o contables. Se usa como cache-aside de la fuente autoritativa — típicamente Postgres o el sistema transaccional de banca — con TTL corto e invalidación explícita en escritura. Se acepta ver datos ligeramente desactualizados, nunca datos incorrectos persistentes".
- **Datos efímeros no se resetean, se dejan morir por TTL**: "En Redis, los datos efímeros no se resetean con código: se dejan morir por TTL. El reseteo activo con cron jobs o updates masivos es un anti-patrón de Redis. Se declara la caducidad al escribir la clave y Redis gestiona el ciclo de vida".
- **TTL como diseño activo, no solo caducidad**: "El TTL en Redis no es solo caducidad, es diseño activo. Rate limiter: TTL fijo desde la creación, no se toca — la clave muere y renace. Sesión de usuario: TTL renovado en cada acceso — la clave sobrevive mientras haya actividad. Mismo mecanismo, dos comportamientos completamente distintos según cuándo se aplica `EXPIRE`".
- **Creación y renovación de TTL son operaciones distintas**: "En Redis, crear una clave con TTL y renovar el TTL de una clave existente son operaciones distintas con comandos distintos. `SET ... EX ... NX` crea con TTL inicial una sola vez, típicamente al login. `EXPIRE` o `GETEX` renuevan el TTL de una clave existente en cada acceso. Confundir los dos lleva a bugs sutiles: sobrescribir sesión al re-crearla, o intentar renovar TTL de una clave inexistente sin darse cuenta".
- **Persistencia por defecto, caducidad opt-in**: "En Redis, la persistencia es la opción por defecto y la caducidad es opt-in. Si guardas una clave sin `EXPIRE` ni flag `EX`, vive para siempre. Al revés de la intuición de quien viene de otras BD. Consecuencia: olvidar el TTL en claves efímeras es una fuga de memoria clásica".
- **Diseño de claves = diseño de schema**: "El diseño de claves en Redis es equivalente al diseño de schema en Postgres: es donde vive la arquitectura de la solución. Meter información identificativa en la clave (una clave por elemento, TTL independiente) o meterla como campos dentro de un Hash (TTL compartido, lectura agrupada del conjunto) son decisiones de diseño con tradeoffs concretos, no detalles menores".
- **`INCR` atómico como base de contadores**: "`INCR` en Redis es atómico. Aunque mil peticiones lleguen simultáneas, cada una incrementa exactamente en 1 sin condiciones de carrera. Es la base de rate limiters y contadores distribuidos: no se necesita transacción, no se necesita lock. Atomicidad heredada del hecho de que Redis es single-threaded".
- **Los casos de uso no son excluyentes**: "Los tres grandes casos de uso de Redis — cache-aside, TTL natural, estado compartido — no son excluyentes. Un dato puede caer en varios simultáneamente. Cuantas más razones convergen, más clara la elección de Redis frente a alternativas".
- **Redis no tiene JOINs**: "Redis no tiene tablas, ni JOINs, ni queries SQL. Si necesitas relacionar cosas, tu aplicación hace dos consultas y las combina en memoria. Redis brilla en accesos por clave conocida ('dame el carrito del usuario 123'); no es la herramienta para queries complejas ('dame todos los usuarios con más de 5 pedidos en el último mes')".

---

## Próxima subsesión

**Bloque 2 — Redis en Docker Compose + `redis-cli` (laboratorio).**

Objetivo: bajar los conceptos de esta sesión a manos, con comandos reales sobre un Redis corriendo.

Plan tentativo:

- Añadir servicio `redis` al `docker-compose.yml` (opción: laboratorio aparte o en `document-analyzer-ai`, a decidir al arrancar).
- Entrar en `redis-cli` desde dentro del contenedor.
- Ejercicios empíricos por cada tipo de dato: `SET/GET` con TTL, `HSET/HGETALL`, `LPUSH/LRANGE`, `SADD/SMEMBERS`, `ZADD/ZRANGE`.
- Verificar `TTL clave` (comando que devuelve segundos restantes) y `EXPIRE` en vivo.
- Ver qué pasa cuando una clave expira mientras la estás mirando.

Densidad cognitiva más baja que la sesión conceptual de hoy. Sesión práctica y visual.

**Bloque 3 (después)**: swap real del `ChatMemory` en `document-analyzer-ai` con Spring Data Redis. **Cierra la deuda pedagógica de Fase 1**.

**Bloque 4 (opcional)**: RDB/AOF, replicación, cache-aside en banca.

---

**Estado emocional al cierre**: sesión larga con corte por día fuera. Aprovechamiento bueno en ambos tramos. Densidad conceptual alta en la parte de la tarde con buena digestión: preguntas de comprobación revelaron aciertos claros ("Redis, TTL y estado compartido" para el carrito) y ambigüedades corregidas en el momento ("hash con usuario + serie + minuto" para múltiples series → colisión detectada y refactorizada a Opción A o B). El "Buffffffff" al arrancar el último ejercicio fue señal correcta de saturación cognitiva; el corte se hizo antes de meter otro concepto encima. Aprendizaje sólido de una tecnología nueva desde cero en una sesión — pedagógicamente eficiente. Errores del asistente reconocidos y corregidos sin softening (orden de proyectos, placeholders opacos, ambigüedad en `GETEX`). Contrato de arco (3-4 sesiones) explícito y claro. Preparado para bloque 2 con cabeza fresca otro día.
