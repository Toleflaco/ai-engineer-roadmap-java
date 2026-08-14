# Sesión Redis part-4 — 2026-08-12 tarde

## Contexto

Bloque 3 del arco Redis: continuación tras el falso arranque de la sesión anterior. Estado de partida: repo en `bc7fa91`, cinco cambios sin commit (pom, application.properties, docker-compose.yml, RedisChatMemoryRepository.java, DocumentAnalyzerAiApplication.java), `document-analyzer-redis` levantado tras `docker compose up redis -d` (Docker Desktop se había reiniciado con Windows). Diagnóstico cerrado ayer: `Message` es interfaz, Jackson no sabe qué clase concreta instanciar al deserializar, decisión lockeada de usar DTO propio `MessageRecord`.

## Objetivo

Implementar el DTO `MessageRecord` con mapeo bidireccional, verificar chat funcional end-to-end incluyendo persistencia a través de reinicio de JVM, implementar `findConversationIds` con `SCAN`, y cerrar el bloque 3 con commits limpios en `origin/main`.

## Hallazgos técnicos

### DTO `MessageRecord` — desacoplamiento persistencia/dominio

`record MessageRecord(String messageType, String text)`. Dos campos. Sin lógica. Java records se deserializan por posición de parámetros con Jackson 3.x sin anotaciones adicionales.

Flujo:
- **Escritura** (`saveAll`): `Message → MessageRecord → JSON string`. Stream con dos `.map()`: primero construye el record, después serializa.
- **Lectura** (`findByConversationId`): `JSON string → MessageRecord → Message`. Jackson deserializa el DTO (paso mecánico); un método `toMessage(record)` con switch expression sobre `messageType` construye la instancia concreta (`UserMessage`, `AssistantMessage`, `SystemMessage`).

⭐⭐⭐ frase de entrevista:
> En el Repository Redis usé un DTO propio para persistir mensajes, en lugar de serializar la interfaz `Message` de Spring AI directamente. El formato JSON en Redis no queda acoplado a las clases internas de la librería y, si el modelo de dominio cambia en una versión futura, la persistencia no se rompe. El coste son ~15 líneas de mapeo bidireccional que se leen de un vistazo.

### Switch expression moderno con `default` seguro

```java
return switch (record.messageType()) {
    case "USER" -> new UserMessage(record.text());
    case "ASSISTANT" -> new AssistantMessage(record.text());
    case "SYSTEM" -> new SystemMessage(record.text());
    default -> throw new IllegalStateException("Unexpected messageType: " + record.messageType());
};
```

`default` lanza excepción con el valor recibido incluido — diagnóstico limpio si Spring AI mete un `TOOL` o cualquier tipo nuevo en el futuro. Nunca fallo silencioso.

### `findConversationIds` con `SCAN`

Implementado con `SCAN` (no `KEYS`). Patrón profesional lockeado.

```java
ScanOptions options = ScanOptions.scanOptions().match(KEY_PREFIX + "*").build();
List<String> ids = new ArrayList<>();
try (Cursor<String> cursor = redis.scan(options)) {
    while (cursor.hasNext()) {
        String fullKey = cursor.next();
        String id = fullKey.substring(KEY_PREFIX.length());
        ids.add(id);
    }
}
return ids;
```

⭐⭐⭐ para entrevista:
> En producción nunca uses `KEYS`, es O(n) y bloquea Redis mientras escanea toda la base de datos. Con 10 millones de claves puedes congelar el servidor durante segundos y parar a todos los clientes. `SCAN` desde Redis 2.8 hace el mismo trabajo iterando por trozos sin bloquear. En Java se expone como `Cursor<String>` que se cierra con try-with-resources.

Detalle sutil: `substring(KEY_PREFIX.length())` en lugar de `substring(0, 13)` — magic number evitado, código sobrevive a cambios futuros del prefijo.

### La cadena de advisors como matryoshkas

Aprendizaje conceptual central de la sesión. Cada advisor en Spring AI es una capa de cebolla con tres partes: fase before, `chain.nextCall(request)` (paso hacia dentro), fase after. La request atraviesa las capas hacia dentro (order menor = más externo), llega al LLM, y la respuesta vuelve por las mismas capas hacia fuera. El orden se invierte al salir (LIFO / stack).

Mismo patrón conceptual que Spring Security `FilterChain`, Servlet `Filter`, middleware de Express.js, interceptors de gRPC. Aprender el patrón aquí lo hace reconocible en todos esos frameworks.

### Bug del `LlmLoggingAdvisor` con `MessageChatMemoryAdvisor`

Diagnosticado empíricamente con `MONITOR`:

- **Con `LlmLoggingAdvisor` en el builder**: primer `RPUSH` tras la respuesta LLM guarda `[user, user, assistant]`. Duplicado del user message.
- **Sin `LlmLoggingAdvisor`**: primer `RPUSH` guarda `[user]`, segundo `RPUSH` guarda `[user, assistant]`. Comportamiento correcto según la doc oficial de Spring AI.

Causa raíz probable (en investigación): interacción entre `getOrder()` del advisor custom y cómo `MessageChatMemoryAdvisor` procesa el request en la cadena. Puede que `MessageChatMemoryAdvisor.add()` interno se llame dos veces cuando hay un advisor adicional con `order=0`.

Decisión pragmática: `LlmLoggingAdvisor` fuera del `ChatController` builder con TODO explícito. El parámetro `loggingAdvisor` se mantiene en el constructor como evidencia de que el bean sigue existiendo. Los endpoints `/analyze` y `/analyze/pdf` usan `ChatClient` distintos y no están afectados. Deuda visible en código, no solo en diario.

### `MONITOR` — herramienta operacional confirmada

Segundo uso de `MONITOR` como lupa de diagnóstico en dos sesiones consecutivas. Sin él:
- Ayer no habríamos descubierto que `RedisChatMemoryRepository` no se ejecutaba.
- Hoy no habríamos descubierto que `LlmLoggingAdvisor` duplica mensajes.

Regla asentada: **cuando algo no cuadra en la integración con Redis, primer paso `MONITOR` en una terminal aparte para ver qué comandos llegan (o no llegan)**.

### `git add -p` con hunk no divisible

Aprendizaje operacional. Al hacer `git add -p pom.xml` para separar el reformateo del pom de la nueva dependencia, git presentó un único hunk enorme (todo el fichero cambiado por whitespace + una dependencia nueva). Responder `y` acepta todo el bloque, `n` lo descarta entero. Para separar, hay que usar `e` (edit manual del patch) — se abre un editor donde se pueden borrar líneas concretas del diff antes de aplicarlo.

Decisión pragmática de sesión: aceptar hunk completo, fusionar reformato + dependencia en un solo commit, mencionar el trade-off en el mensaje. Alternativa profesional (para próxima vez): usar `git add -p` con `e`, o bien preparar el fichero en dos pasos separados (reformatear + commit + añadir dependencia + commit).

## Estado del repo al cerrar

- Rama `main`, HEAD en `fb1db78`, sincronizada con `origin/main`.
- Dos commits nuevos:
  - `7c676cb feat(chat): persist chat memory in Redis via custom ChatMemoryRepository` (6 ficheros, 227 líneas +, 95 -).
  - `fb1db78 chore(chat): disable LlmLoggingAdvisor pending investigation` (1 fichero, 7 líneas +, 2 -).
- Working tree clean.
- Contenedor `document-analyzer-redis` levantado.

## Prueba empírica de cierre

Tres curls consecutivos con `conversationId=demo-1`, con reinicio de la JVM entre el segundo y el tercero. LLM respondió correctamente el nombre en los tres turnos. `LRANGE chat:memory:demo-1 0 -1` al final: 6 mensajes en orden cronológico (3 turnos completos), formato JSON limpio con solo `messageType` y `text`, sin duplicados.

**La divergencia "in-memory not Redis" de Fase 1 del roadmap AI Engineer queda cerrada.**

## Deuda técnica al cerrar

- **Investigación `LlmLoggingAdvisor` × `MessageChatMemoryAdvisor`**: causa del duplicado. Probablemente relacionado con `getOrder()` o con cómo Spring AI re-envuelve requests en la cadena de advisors. Prueba concreta: cambiar `order` a un valor alto (500-1000) y ver si desaparece el duplicado.
- **`MULTI/EXEC` para atomicidad de `saveAll`**: hoy `DEL` + `RPUSH` + `EXPIRE` van por separado. Interleaving posible entre dos peticiones concurrentes al mismo `conversationId`. En Java con Spring Data Redis se hace vía `SessionCallback`.
- **Mover `TTL` (y en su día `WINDOW_SIZE` si vuelve) a `application.properties`**.
- **Comentarios narrativos en `saveAll`** (`// convertir messages a...`, `// rightPushAll...`, `// expire`): pedagógicos, decidí mantenerlos hoy pero son candidatos a borrar en próximo refactor de calidad.
- **Actualizar README** para reflejar Redis integrado (pendiente para mañana o próximo turno).

## Estado emocional y candidaturas

Sin novedades desde ayer (tres rechazos vía ATS). Sensación gestionable al no acumularse más noticias negativas. La victoria técnica de hoy (chat que recuerda tras reinicio + dos commits limpios pusheados) equilibra el ruido de agosto en el mercado laboral.

Sesión de dos horas efectivas, densidad alta. Pescadería confirmada como turno de mañana (los miércoles Tole tiene la tarde entera). Fatiga puntual normal al final tras diagnóstico intenso del `LlmLoggingAdvisor` y las vueltas del `git add -p`. Cierre honesto.

## Bloque 4 — opcional, sin arrancar

Los tres temas quedan disponibles cuando haya energía y contexto:

1. Persistencia RDB vs AOF: qué son, cómo se configuran, cuándo elegir cada uno.
2. Replicación básica master-replica para HA.
3. Cache-aside pattern en banca: consultas de saldos cacheadas con TTL corto, invalidación en escritura. Frase ⭐⭐⭐ ya redactada en el diario del bloque 1.

Densidad esperada: 45-60 minutos si se hace conceptual sin código propio, o 90-120 min con un mini laboratorio en `~/proyectos/redis-lab/`.
