# Sesión Redis part-3 — 2026-08-11 tarde

## Contexto

Bloque 3 del arco Redis: integración con Spring Boot en `document-analyzer-ai`. Continuación tras descubrir en la sesión anterior (part-2 no, la sesión sin diario del 2026-08-10 tarde donde arrancó el bloque 3) que `implements ChatMemory` era la interfaz equivocada — Spring instanciaba la clase pero el `MessageChatMemoryAdvisor` nunca llamaba a sus métodos.

Estado de partida: repo en `bc7fa91`, contenedor `document-analyzer-redis` levantado, `RedisChatMemory.java` sin trackear con implementación incorrecta (`implements ChatMemory`), `pom.xml` y `application.properties` con cambios sin commit.

## Objetivo

Refactorizar la clase a `implements ChatMemoryRepository`, ajustar el bean `ChatMemory` para inyectar el Repository custom, e implementar los métodos con `RedisTemplate`. Verificar empíricamente con `MONITOR` que Spring AI llama al Repository custom en lugar del `InMemoryChatMemoryRepository` por defecto.

## Hallazgos técnicos

### Interfaz correcta confirmada — `ChatMemoryRepository`

Verificado con `unzip -l ~/.m2/repository/org/springframework/ai/spring-ai-model/2.0.0/spring-ai-model-2.0.0.jar | grep ChatMemory`. La interfaz vive en `org.springframework.ai.chat.memory.ChatMemoryRepository`, mismo package que `MessageWindowChatMemory` y `InMemoryChatMemoryRepository`. Cuatro métodos:

- `void saveAll(String conversationId, List<Message> messages)` — **reemplaza todo el histórico** de la conversación, no añade.
- `List<Message> findByConversationId(String conversationId)` — lee todo el histórico.
- `void deleteByConversationId(String conversationId)` — borra la conversación entera.
- `List<String> findConversationIds()` — lista todos los IDs de conversación en el storage.

No hace falta añadir dependencias al `pom.xml` — la interfaz ya está en el classpath vía `spring-ai-starter-model-anthropic`.

### ⭐⭐⭐ Separación de responsabilidades `ChatMemory` / `ChatMemoryRepository`

Es el hallazgo arquitectónico central de la sesión. Spring AI 2.0 separó explícitamente dos ejes ortogonales:

- **`ChatMemory`** (política de retención): decide cuánto guardar, cómo recortar, si resumir, si comprimir. Implementación built-in: `MessageWindowChatMemory` con ventana fija de N mensajes.
- **`ChatMemoryRepository`** (persistencia): CRUD puro sobre el storage. No sabe nada de ventana ni política.

Consecuencias prácticas:

1. Cambiar de ventana 10 a ventana 20 no toca el Repository.
2. Cambiar de Redis a Postgres no toca el `ChatMemory`.
3. Añadir una `SummarizingChatMemory` que resuma mensajes viejos con el LLM no requiere reescribir el storage.

Es el mismo patrón que Spring Data lleva 15 años aplicando: `Repository` no sabe de reglas de negocio, `Service` no sabe de SQL. Frase de entrevista lista:

> Spring AI 2.0 separó `ChatMemory` (política de retención) de `ChatMemoryRepository` (persistencia). Al implementar Redis, solo tocas el Repository, la política de ventana vive fuera y se puede cambiar sin tocar código de Redis.

### `saveAll` es replace, no append

De la doc oficial: "Replaces all the existing messages for the given conversation ID with the provided messages". Consecuencias en la implementación con Redis:

- `LPUSH` iterativo NO sirve — acumularía en cada llamada.
- Patrón correcto: `DEL` + `RPUSH` + `EXPIRE`.
- `LTRIM` innecesario — `MessageWindowChatMemory` ya recorta a `maxMessages` antes de invocar `saveAll`. Tú siempre recibes como mucho N mensajes.

Flujo interno confirmado por el stacktrace: cuando llega `/chat`, `MessageWindowChatMemory.add()` primero llama a `findByConversationId` para leer el histórico, añade los nuevos `[user, assistant]`, aplica la ventana (recorta si excede `maxMessages`), y llama a `saveAll` con la lista final completa.

### Bean `ChatMemory` modificado

Antes:
```java
@Bean
ChatMemory chatMemory() {
    return MessageWindowChatMemory.builder()
            .maxMessages(10)
            .build();
}
```

Después:
```java
@Bean
ChatMemory chatMemory(ChatMemoryRepository chatMemoryRepository) {
    return MessageWindowChatMemory.builder()
            .chatMemoryRepository(chatMemoryRepository)
            .maxMessages(10)
            .build();
}
```

Spring resuelve la inyección automáticamente porque el `@Component RedisChatMemoryRepository` es el único bean de tipo `ChatMemoryRepository` en el contexto — sustituye al `InMemoryChatMemoryRepository` por defecto.

### Diseño de claves

`chat:memory:{conversationId}` como Redis List. Cada elemento es el JSON del `Message`. Prefijo `chat:memory:` para agrupar lógicamente y permitir `SCAN chat:memory:*` en `findConversationIds` (pendiente).

Orden: `RPUSH` para escribir + `LRANGE 0 -1` para leer → orden cronológico natural. Si se hubiera usado `LPUSH` habría salido en orden inverso.

### Mapeo Spring Data Redis → comandos Redis

| Comando Redis | API Spring Data Redis |
|---|---|
| `DEL key` | `redis.delete(key)` |
| `RPUSH key v1 v2 ...` | `redis.opsForList().rightPushAll(key, values)` |
| `LRANGE key 0 -1` | `redis.opsForList().range(key, 0, -1)` |
| `EXPIRE key seconds` | `redis.expire(key, duration)` — Spring lo traduce a `PEXPIRE` con milisegundos |

Nota: `range()` puede devolver `null` según el contrato de Spring Data Redis, no solo lista vacía. Hoy se comprobó con `isEmpty()`, hay que endurecer a `jsons == null || jsons.isEmpty()`.

### `MONITOR` como herramienta empírica

⭐⭐⭐ Aprendizaje operacional. Comandos capturados en `MONITOR` tras el primer `curl` al `/chat`:

```
LRANGE "chat:memory:demo-1" "0" "-1"        <- MessageWindowChatMemory lee histórico
LRANGE "chat:memory:demo-1" "0" "-1"        <- relee (patrón interno del advisor)
DEL "chat:memory:demo-1"                     <- saveAll empieza (nuestro DEL)
RPUSH "chat:memory:demo-1" "{\"media\":[],\"messageType\":\"USER\",...}"
PEXPIRE "chat:memory:demo-1" "86400000"     <- TTL en ms (Spring traduce Duration.ofHours(24))
LRANGE "chat:memory:demo-1" "0" "-1"        <- relectura para responder
```

Sin `MONITOR` habríamos vuelto a asumir arquitectura en lugar de verificarla. Es la lupa que revela empíricamente si el código toca Redis.

### Otros hallazgos operacionales

- Cuando se duda de una interfaz, dejar que IntelliJ la refactorice con "Implement Methods" y comparar con la documentación oficial. Doble verificación cruzada — hoy los cuatro stubs de IntelliJ coincidieron exactamente con la doc.
- Stacktraces se leen enteros hasta encontrar la línea de código propio (`RedisChatMemoryRepository.java:60`). Ahí está la información útil, no en el ruido de framework.

## Falso arranque diagnosticado

Segundo curl (o cualquier flujo donde haya que leer histórico previo) peta con:

```
tools.jackson.databind.exc.InvalidDefinitionException: 
Cannot construct instance of `org.springframework.ai.chat.messages.Message` 
(no Creators, like default constructor, exist): 
abstract types either need to be mapped to concrete types, 
have custom deserializer, or contain additional type information
```

**Causa**: `Message` es interfaz en Spring AI. Al serializar con `writeValueAsString(mensaje)` sale un JSON con campo `messageType` (`"USER"`, `"ASSISTANT"`, `"SYSTEM"`), pero Jackson por defecto no lo interpreta como discriminador para elegir la clase concreta a instanciar. `readValue(json, Message.class)` no sabe si debe construir `UserMessage`, `AssistantMessage` o `SystemMessage`.

**Decisión lockeada para próxima sesión**: DTO propio `MessageRecord(String messageType, String text)` con mapeo bidireccional en el Repository. Ventajas: sin acoplamiento con clases internas de Spring AI (si cambian los campos internos, Redis sigue funcionando), formato JSON limpio y estable, patrón "DTO de persistencia distinto del dominio" que es clásico en microservicios y banca. Alternativas descartadas: `activateDefaultTyping` en `ObjectMapper` (riesgo de seguridad, CVE clásicos de jackson-databind) y custom `JsonDeserializer<Message>` (más código de infraestructura, menos didáctico).

## Estado del repo al cerrar

- Rama `main`, HEAD en `bc7fa91` (sincronizado con `origin/main`).
- **Sin commits nuevos** — código no funciona end-to-end.
- Cambios sin commit:
  - `pom.xml` (dependencia `spring-boot-starter-data-redis`).
  - `application.properties` (host y port de Redis).
  - `docker-compose.yml` (nuevo, servicio `redis` + servicio `app` con profile `full-stack`).
  - `src/main/java/dev/toleflaco/document_analyzer_ai/chat/RedisChatMemoryRepository.java` (nuevo, tres métodos implementados, `findConversationIds` como stub).
  - `src/main/java/dev/toleflaco/document_analyzer_ai/DocumentAnalyzerAiApplication.java` (bean `ChatMemory` modificado para inyectar `ChatMemoryRepository`).
- Contenedor `document-analyzer-redis` levantado, se deja vivo para retomar mañana sin fricción de arranque.

## Deuda técnica al cerrar

- **DTO `MessageRecord` + mapeo bidireccional** en `RedisChatMemoryRepository` (bloqueo actual, siguiente paso).
- **`findConversationIds()`** sin implementar. Usar `SCAN` con `ScanOptions.scanOptions().match("chat:memory:*").build()`, iterar el `Cursor<String>`, quitar el prefijo. Nunca `KEYS` en producción.
- **`MULTI/EXEC`** para atomicidad de `saveAll` (evita interleaving de dos peticiones concurrentes al mismo `conversationId` — hoy `DEL` y `RPUSH` van por separado).
- Mover `WINDOW_SIZE` (obsoleto) y `TTL` a `application.properties`.
- Sustituir `System.out.println` del constructor por SLF4J.
- Endurecer `findByConversationId` contra `null` de `range()`: `if (jsons == null || jsons.isEmpty())`.
- Renombrar constante `WINDOW_SIZE` — ya no se usa porque la ventana la aplica `MessageWindowChatMemory`. Borrar directamente.
- Manejo real de `JacksonException` en `saveAll` — hoy queda propagada como unchecked. Decidir política de logging + fallback o re-throw.

## Contexto para próxima sesión

Al arrancar:

1. `git status` — debe mostrar los mismos cinco cambios sin commit.
2. `docker compose ps` — `document-analyzer-redis` debería seguir vivo.
3. Confirmar que la app arranca sin errores (Spring instancia `RedisChatMemoryRepository`, `>>> RedisChatMemory created by Spring <<<` en el log).

Plan de trabajo:

1. Crear `MessageRecord` como Java record con dos campos: `messageType` (String) y `text` (String).
2. En `saveAll`: mapear `Message → MessageRecord` antes de serializar. Usar `message.getMessageType().name()` para el discriminador y `message.getText()` para el contenido.
3. En `findByConversationId`: deserializar a `MessageRecord`, después mapear a `Message` con un `switch` sobre `messageType`:
   - `"USER"` → `new UserMessage(record.text())`.
   - `"ASSISTANT"` → `new AssistantMessage(record.text())`.
   - `"SYSTEM"` → `new SystemMessage(record.text())`.
4. Verificar con `MONITOR` que el JSON serializado ahora es limpio (solo `messageType` + `text`, sin `media`/`metadata` internos).
5. Prueba end-to-end de tres pasos: curl 1 ("me llamo Tole"), curl 2 ("cuál es mi nombre") — debe recordar. Reiniciar app. Curl 3 mismo `conversationId` — debe seguir recordando desde Redis.
6. Si todo funciona, implementar `findConversationIds` con `SCAN`.
7. Refactor de atomicidad con `MULTI/EXEC` como paso separado.
8. Commits bilingües limpios y push.
9. Actualizar README quitando/rebajando la divergencia "in-memory not Redis".

Densidad restante estimada: 60-90 minutos si todo va limpio. Sesión tarde de un día laborable es adecuada.

## Estado emocional y candidaturas

Tres rechazos vía ATS más hoy (van siete en cuatro días contando los del sábado). Interpretación: filtros automáticos por keywords en agosto, no señal de mercado. Canales activos (Hays / CAS Training / Netcheck) siguen pendientes de respuesta o segunda ronda, ninguno cerrado. Realidad operacional: procesos de banca/consultoría en España se ralentizan hasta primera semana de septiembre. Lo que se controla hoy es el roadmap; el resto es paciencia estratégica.

Sesión cerrada con sensación de "no acabar", corregida con inventario: hoy se resolvió el bloqueo estructural del arco (interfaz correcta, bean modificado, verificación empírica con `MONITOR`). Lo que queda mañana es decisión arquitectónica limpia (DTO) + tecleo mecánico. Toda la parte pensante quedó cerrada.
