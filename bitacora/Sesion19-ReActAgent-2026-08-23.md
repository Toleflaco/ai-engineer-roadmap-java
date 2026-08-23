# Sesión 19 — 2026-08-23 (ReAct agent parte 3, bucle mínimo funcional end-to-end)

**Continuación del arco iniciado en S16-S17 (conceptual) y S18 (bootstrap + handshake MCP). Fase 2.2 del roadmap AI Engineer — Proyecto 2: `erp-purchasing-agent`. Sesión de código real: diseño socrático completo del `ReActAgent`, bucle ReAct explícito camino A puro escrito línea a línea, endpoint REST expuesto, test de humo empírico exitoso, primer commit del bucle pusheado a GitHub.**

Duración efectiva: ~4h, domingo por la mañana temprano (arrancamos a las 6:45, cerramos hacia las 11:00). Energía sostenida, sin fatiga significativa. Sesión con densidad alta de decisiones de diseño socráticas, dos bugs sutiles diagnosticados empíricamente, y un cierre en verde con `curl` real devolviendo respuesta estructurada del agente.

## Repo state al cerrar

- **`~/proyectos/erp-purchasing-agent/`**: HEAD `0f192af`, working tree clean, pusheado a `origin/main`. Cuatro archivos nuevos versionados en tres packages (`agent/`, `controller/`, `dto/`). Sesión con código Java propio por primera vez en este proyecto.
- **`~/proyectos/erp-mcp-server/`**: HEAD `ff35c28`, working tree clean. Levantado durante la sesión (`docker compose up -d`), sigue arriba al cierre (opcional pararlo con `docker compose down`).
- **`~/proyectos/document-analyzer-ai/`**: HEAD `8e03afb`, working tree clean. No tocado.

Contenedores: `erp-app` y `erp-postgres` levantados. Proceso Spring Boot del agente parado tras cierre de sesión con `Ctrl+C`.

## Commits generados en la sesión

- **`0f192af`** — `feat(agent): implement minimal ReAct loop with MCP tool integration`
  - Cuatro archivos nuevos: `ReActAgent.java`, `AgentController.java`, `AgentRunRequest.java`, `AgentRunResponse.java`.
  - 89 insertions.
  - Mensaje bilingüe con separador `---`, cuerpo español sin tildes/ñ.
  - Pusheado a `origin/main`.

## Conceptos cubiertos

### Diseño socrático del contrato público de `ReActAgent`

Diseño en pizarra antes de tocar el editor. Contrato lockeado:

```
Package:    dev.toleflaco.erp_purchasing_agent.agent
Clase:      ReActAgent
Anotación:  @Service
Firma:      public String run(String prompt)
Dependencias inyectadas por constructor:
  - ChatModel chatModel
  - List<McpSyncClient> mcpClients
```

Cada decisión razonada antes de teclear. La superficie pública se define pensando en cómo la llama el resto de la aplicación (el `RestController`), no pensando en qué hace por dentro.

### Separación `@Service` vs `@RestController` — razón arquitectónica

Tole propuso inicialmente `@RestController` para el `ReActAgent`, confundiendo capas. Corrección: `@RestController` es adaptador HTTP puro (traduce HTTP↔Java), la lógica de negocio va en `@Service`.

**Tres razones concretas** de por qué la separación importa (más allá de "SRP" abstracto):

1. **Testeo unitario sin infraestructura web**: `@Service` permite instanciar con mocks y probar sin `@SpringBootTest` ni Tomcat.
2. **Reutilización desde otros puntos de entrada**: mismo bean invocable desde `@KafkaListener`, `@Scheduled`, `CommandLineRunner` sin duplicar código.
3. **Ecosistema Spring transversal**: `@Transactional`, `@Async`, `@Retryable`, timeouts, cancelación, sólo aplican a beans de servicio. El thread de Tomcat en un `@RestController` está atado al ciclo de vida de la request.

**Regla lockeada**: en Spring Boot, capa HTTP y capa de lógica de negocio siempre en clases distintas. El controller inyecta el service, nunca contiene el service.

### `ChatModel` vs `ChatClient` — camino A puro

Decisión pedagógica lockeada del prompt de continuación. Aclarada empíricamente:

- **`ChatClient`**: API de alto nivel de Spring AI. Fluent (`.prompt().user(...).call().content()`). Devuelve `String` directamente. Si le pasas tools con `.tools(...)`, hace el bucle ReAct internamente. **Es camino B camuflado** — el bucle desaparece de vista.
- **`ChatModel`**: API de bajo nivel. Trabajas con `Prompt` de entrada y `ChatResponse` de salida. TÚ decides qué hacer con la respuesta. **Es lo que necesita camino A**.

Tole inicialmente eligió `ChatClient` con justificación "más control", que es exactamente al revés. Corrección: `ChatClient` = ergonomía, `ChatModel` = control. Refactorizado a `ChatModel` antes de teclear.

**Regla lockeada**: en Spring AI 2.0.0, si quieres ver y controlar el bucle ReAct explícitamente, inyectas `ChatModel`. Si prefieres ergonomía y aceptas que Spring haga el bucle por ti, `ChatClient`.

### Interfaz `ChatModel` vs implementación concreta `AnthropicChatModel`

Elegido inyectar la interfaz `ChatModel`. Justificación de Tole ("por si cambiamos de modelo") corregida: es especulativa (YAGNI). Las razones **verdaderamente defendibles**:

1. **Testabilidad**: `Mockito.mock(ChatModel.class)` es trivial sobre interfaz; sobre clase concreta requiere motor más pesado.
2. **Coherencia con Spring**: convención del framework es tipar por interfaz (`JdbcTemplate`, `RestTemplate`, `RedisTemplate` siguen el mismo patrón).
3. **Cast explícito si hace falta lo específico**: si algún día necesitas prompt caching o extended thinking de Anthropic, castea `(AnthropicChatModel) chatModel`. Para el bucle ReAct estándar, la interfaz basta.

**Regla lockeada**: **prefiere la interfaz sobre la implementación concreta cuando NO necesites nada específico de la implementación**. El motivo principal es testabilidad y coherencia con el ecosistema Spring, no un futuro cambio hipotético de proveedor.

### Dos operaciones del cliente MCP: `tools/list` y `tools/call`

Tole identificó vía Sócrates las dos operaciones distintas contra el `erp-mcp-server`:

1. **`tools/list`**: obtener catálogo. UNA vez al arrancar el bucle (o cacheable al arrancar la app).
2. **`tools/call`**: ejecutar tool concreta. N veces, una por iteración donde el LLM decide invocar tools.

**Frase lockeada**: `tools/list` es stateless y se hace UNA vez por bucle. `tools/call` se hace N veces, una por cada iteración donde el LLM decide invocar una tool.

### Bean autoconfigurado por `spring-ai-starter-mcp-client`: `List<McpSyncClient>`

El starter autoconfigura una **lista** de `McpSyncClient`, uno por cada conexión declarada en el `application.yml`. En nuestro caso la lista tiene UN elemento (`erp-server`), pero la firma correcta del constructor es siempre `List<McpSyncClient>` para no romperse cuando se añadan más servidores.

Cada `McpSyncClient` expone:
- `listTools()` → `ListToolsResult` (para `tools/list`).
- `callTool(CallToolRequest)` → resultado (para `tools/call`).

### `SyncMcpToolCallbackProvider` — helper oficial para el mapper MCP ↔ Spring AI

El formato de `Tool` del MCP SDK (`io.modelcontextprotocol.spec.McpSchema`) no es el mismo que el formato `ToolCallback` de Spring AI (`org.springframework.ai.tool`). Necesitas un adapter.

Decisión pedagógica lockeada: **Opción H (helper oficial)** en vez de Opción P (mapper a mano).

- **Opción P (puro)**: escribes 40-60 líneas de mapper adaptando `Tool` (MCP) → `ToolDefinition` (Spring AI). Ese código no enseña ReAct, enseña conversión de formatos.
- **Opción H (helper)**: `new SyncMcpToolCallbackProvider(mcpClients).getToolCallbacks()` te devuelve `ToolCallback[]` listos para pasar al `ChatModel`. 2-3 líneas.

**Regla lockeada — aclaración crítica**: **camino A = control del bucle. NO significa reimplementar todos los helpers de Spring AI a mano**. Delegar el mapper aburrido a un helper oficial sigue siendo camino A porque el `while` sigue siendo tuyo.

### Cambio importante en Spring AI 2.0.0: `internalToolExecutionEnabled` eliminado

Descubrimiento vía documentación oficial. En 2.0.0, la propiedad `internalToolExecutionEnabled` fue eliminada. Cuando invocas `chatModel.call(prompt)` directamente, las tools **nunca se ejecutan automáticamente**. Es siempre responsabilidad del cliente.

Consecuencia: si trabajas con `ChatModel` directo y quieres ejecución automática de tools, necesitas ir a `ChatClient` (y por tanto a camino B). Si quieres camino A, gestionas el ciclo con `ToolCallingManager` a mano.

**Simplificación real**: elimina un flag oculto y hace el contrato explícito.

### `ToolCallingManager` — user-controlled loop pattern

Utilidad stateless de Spring AI 2.0.0. Recibe un `Prompt` y una `ChatResponse` con `tool_use`, ejecuta las tools por dentro (buscando el `ToolCallback` correcto por nombre y llamándolo), y devuelve un `ToolExecutionResult` con el `conversationHistory()` ya actualizado con los `ToolResponseMessage` añadidos.

Patrón canónico documentado en `docs.spring.io/spring-ai/reference/api/chat/anthropic-chat.html`:

```java
ToolCallingManager toolCallingManager = ToolCallingManager.builder().build();
AnthropicChatOptions options = AnthropicChatOptions.builder()
    .toolCallbacks(toolCallbacks)
    .build();
Prompt prompt = new Prompt(messages, options);
ChatResponse response = chatModel.call(prompt);
while (response.hasToolCalls()) {
    ToolExecutionResult result = toolCallingManager.executeToolCalls(prompt, response);
    prompt = new Prompt(result.conversationHistory(), options);
    response = chatModel.call(prompt);
}
return response.getResult().getOutput().getText();
```

Este patrón **sigue siendo camino A**: el `while` es del cliente, los guardrails futuros van dentro del `while` como `if`s, `ToolCallingManager` solo hace el paso mecánico de ejecutar las tools y construir el historial.

### `while (condicion)` vs `while (true)` — patrón loop-and-a-half

Inicialmente pensado `while (true)` con salida por `return` desde dentro. Con `ToolCallingManager` disponible, el patrón canónico es `while (response.hasToolCalls())` que queda más limpio.

**Regla lockeada — patrón loop-and-a-half**: **en un `while (condicion(x))`, la variable `x` tiene que estar inicializada ANTES de entrar (para que la primera evaluación tenga algo que leer) y actualizada AL FINAL de cada iteración (para que la siguiente evaluación tenga la versión nueva)**.

Consecuencia práctica: la primera llamada al LLM (`chatModel.call(currentPrompt)`) va **fuera** del `while`, y cada iteración termina con `response = chatModel.call(currentPrompt)`. Bonus gratis: si el LLM decide no llamar tools en la primera llamada, no entramos al `while` en absoluto.

### API de LLMs stateless — la sesión vive en el cliente

Cada `chatModel.call(prompt)` es stateless desde el punto de vista del proveedor. El `ChatModel` no recuerda que en la llamada anterior le pasaste tools ni el historial. Cada request va cargado con todo lo necesario.

**Regla lockeada**: **la API de LLMs es stateless. Cada request va cargado con: (a) todo el historial de la conversación, (b) todas las tools disponibles, (c) las opciones del modelo. No hay "sesión" en el proveedor — la sesión la mantienes tú en el cliente reconstruyendo el `Prompt` en cada iteración**.

Es la misma lógica que Tole implementó en `document-analyzer-ai` con `RedisChatMemoryRepository`: el "recordar" no lo hace Anthropic, lo haces tú serializando y deserializando desde Redis. Aquí el "recordar" lo hace la variable `messages` (historial) y `options` (tools) viviendo en memoria durante `run()`.

### Historial intra-ejecución vs memoria inter-ejecución — scopes distintos

Tole preguntó si "puede haber más llamadas antes de que el agente entre en escena". Aclaración importante:

- **Historial intra-ejecución** (dentro del `while` del ReAct): la lista `messages` que crece a lo largo de las iteraciones. Se descarta cuando `run()` retorna.
- **Memoria inter-ejecución** (multi-turn conversacional): persistencia externa (Redis, DB) para que dos `POST /agent/run` del mismo usuario recuerden el contexto compartido. Requiere `ChatMemory` + `conversationId`.

**Decisión de scope lockeada para `erp-purchasing-agent` v1**: cada `POST /agent/run` es una sesión fresca sin memoria de peticiones anteriores. No hay `ChatMemory` inyectado. La memoria conversacional se añade en versión 2 (deuda #8, ver más abajo) cuando la v1 esté cerrada con guardrails y HITL.

### `AnthropicChatOptions` vs `ToolCallingChatOptions` — trade-off de portabilidad

**Bug diagnosticado empíricamente durante el test de humo**. La primera versión del código usaba `ToolCallingChatOptions` (interfaz genérica del package `org.springframework.ai.model.tool`). Resultado: el LLM respondía como si no tuviera tools, sin ejecutar ninguna.

Diagnóstico: `AnthropicChatModel.call(prompt)` en Spring AI 2.0.0 **solo extrae los `toolCallbacks` si el `ChatOptions` del prompt es del tipo específico `AnthropicChatOptions`**. Con un `DefaultToolCallingChatOptions` genérico, el chat model no lo reconoce y manda la petición sin tools.

Fix: cambiar `ToolCallingChatOptions.builder()` por `AnthropicChatOptions.builder()`. Una línea.

**Trade-off asumido y defendible en entrevista**: **en Spring AI 2.0.0 la portabilidad entre proveedores llega hasta la interfaz `ChatModel`, pero las `ChatOptions` son específicas del proveedor porque cada uno soporta parámetros propios**. Cambiar de proveedor requiere cambiar la línea de las options; el resto del código no se toca. Una línea contra cambio de proveedor es aceptable.

### DTOs con `record` sobre `Map<String, ?>` — cinco razones concretas

Elección lockeada: DTOs (`AgentRunRequest`, `AgentRunResponse`) como records, no `Map<String, String>`. Razones concretas más allá de "es lo correcto":

1. **Contrato explícito**: keys visibles en el código, IntelliJ autocompleta.
2. **Validación**: `@NotBlank String prompt` funciona sobre record, no sobre Map.
3. **OpenAPI útil**: springdoc genera schema real desde tipos concretos.
4. **Tipado**: refactor asistido si cambian los campos.
5. **Tests limpios**: `MockMvc` más legible con DTOs tipados.

**Regla lockeada**: **DTOs con `record` sobre `Map<String, ?>` para request/response bodies. Cinco razones concretas, no una general**.

## ⭐⭐⭐ Frases para entrevista

- **Sobre separación de capas**:
  > "La lógica en `@Service` te habilita el ecosistema Spring transversal — `@Transactional`, `@Async`, `@Retryable`, timeouts, testing sin contexto web — que el `@RestController` no puede aprovechar porque su ciclo de vida está atado al thread de Tomcat."

- **Sobre la interfaz vs implementación**:
  > "Prefiere la interfaz sobre la implementación concreta cuando NO necesites nada específico de la implementación. El motivo principal es testabilidad y coherencia con el ecosistema Spring, no un futuro cambio hipotético de proveedor."

- **Sobre camino A**:
  > "Camino A = control del bucle. NO significa reimplementar todos los helpers de Spring AI a mano. La granularidad del control se mide en decisiones arquitectónicas (cuándo salgo, qué guardrails aplico, cómo persisto), no en si escribo el mapper de tool calls a bajo nivel."

- **Sobre API stateless de LLMs**:
  > "La API de LLMs es stateless. Cada request va cargado con: todo el historial de la conversación, todas las tools disponibles, las opciones del modelo. No hay 'sesión' en el proveedor — la sesión la mantengo yo en el cliente reconstruyendo el `Prompt` en cada iteración."

- **Sobre el bucle ReAct explícito**:
  > "El bucle ReAct explícito son 21 líneas: preparar historial y tools, primera llamada al LLM fuera del bucle para inicializar la condición, iterar mientras el LLM pida tools invocándolas contra el servidor MCP y reconstruyendo el prompt con el historial acumulado, y devolver el texto cuando el LLM responde sin más tool calls. Cada llamada al LLM es stateless, la sesión la mantengo yo en el cliente."

- **Sobre trade-off de portabilidad en Spring AI 2.0.0**:
  > "En Spring AI 2.0.0 la portabilidad entre proveedores llega hasta la interfaz `ChatModel`, pero las `ChatOptions` son específicas del proveedor porque cada uno soporta parámetros propios. Cambiar de proveedor requiere cambiar la línea de las options; el resto del código no se toca."

- **Sobre proyectos de portfolio**:
  > "Cada proyecto de portfolio se justifica por lo que enseña, no por lo que implementa. Un chatbot suelto enseña 'sé usar la API de OpenAI'. Un agente con memoria persistente resolviendo un caso de negocio enseña arquitectura."

- **Sobre patrón loop-and-a-half**:
  > "En un `while (condicion(x))`, la variable `x` tiene que estar inicializada ANTES de entrar (para que la primera evaluación tenga algo que leer) y actualizada AL FINAL de cada iteración (para que la siguiente evaluación tenga la versión nueva). Es el patrón loop-and-a-half en su forma limpia."

## Bugs diagnosticados empíricamente durante la sesión

### Bug 1 — Import equivocado de `@RequestBody`

IntelliJ ofreció dos opciones para `@RequestBody`: `org.springframework.web.bind.annotation.RequestBody` (Spring, el correcto) y `io.swagger.v3.oas.annotations.parameters.RequestBody` (Swagger/OpenAPI, documentación pura). Tole escogió el de Swagger. Compila sin error. En runtime, Spring no deserializa el body al DTO, llega con `prompt = null`, y estalla con NPE al invocar `request.prompt()`.

Cazado en review pre-arranque. Corregido antes de probar.

**Regla reforzada**: cuando IntelliJ ofrece varios imports para el mismo nombre, para y verifica el package. Es la misma clase de bug que con `Message` (colisión Spring AI / Anthropic SDK / MCP SDK). Bug silencioso, compila, falla raro en runtime.

### Bug 2 — Constructor con parámetro fantasma

IntelliJ generó constructor con dos parámetros del mismo tipo:

```java
public AgentController(ReActAgent agent, ReActAgent reActAgent) {
    this.reActAgent = reActAgent;
}
```

Compila y funciona (Spring inyecta el mismo singleton dos veces), pero es código roto. Un review senior lo tumba. Corregido a un solo parámetro.

### Bug 3 — Tools no llegando al LLM (H2)

Test de humo devolvió respuesta del LLM diciendo "no tengo acceso a tu base de datos, dame los datos". Cero actividad MCP en logs durante la request. Diagnóstico por hipótesis:

- **H1**: `SyncMcpToolCallbackProvider` no descubre tools. Verificada con `System.out.println` mostrando `toolCallbacks.length` y nombres. **13 tools presentes**. H1 descartada.
- **H2**: `ToolCallingChatOptions` genérico ignorado por `AnthropicChatModel`. Confirmada por descarte + documentación oficial. Fix: `AnthropicChatOptions.builder()`.

**Regla reforzada**: "arranca sin error ≠ funciona". El primer `curl` es la verificación real, no el `Started ...` de Spring Boot.

## Estado de la app y roadmap

- **`erp-purchasing-agent`**: bucle ReAct mínimo funcional end-to-end. 4 archivos, 89 líneas Java propias. Verificado empíricamente con escenario canónico "reposición de stock" (READ-only): el agente llamó `listLowStockProducts` + `getSupplier` en paralelo, sintetizó respuesta estructurada por categorías con proveedores identificados. Sin guardrails, sin logging por iteración, sin observabilidad estructurada. Primer commit del bucle en `origin/main`.
- **`erp-mcp-server`**: sin cambios (HEAD `ff35c28`). 13 tools disponibles, todas descubiertas por el cliente MCP del agente.
- **`document-analyzer-ai`**: sin cambios (HEAD `8e03afb`).
- **Roadmap AI Engineer, Fase 2.2 — Proyecto 2 (`erp-purchasing-agent`)**: core ReAct resuelto. Siguientes hitos: `LlmLoggingAdvisor` para observabilidad por iteración, guardrails formales, HITL, README profesional.

## Correcciones y aprendizajes de proceso

- **Sobredosis de información con el patrón de introspección**: en un momento sugerí a Tole aplicar el patrón `jar tf` + `jar xf` + `cat` sobre el jar del starter MCP para descubrir el bean autoconfigurado. Tole marcó explícitamente "no sé lo que me estás pidiendo, me ha descolocado, cada vez que avanzamos más creo que sé menos". Detección correcta de sobredosis. Rebobiné, ofrecí plan (a) — te digo el nombre directamente — vs (b) — te acompaño a mirar la doc. Tole eligió (b). Cuando la búsqueda concreta también le atascó ("no sé buscar esa información, joder"), salté a (a). **Aprendizaje**: cuando el terreno es genuinamente nuevo, meter una decisión por mensaje, no tres. La sensación de "sé menos" es señal de dosis mal calibrada, no de aprendizaje deficiente. **Nueva regla lockeada**: cuando el terreno es nuevo, en próximos bloqueos similares dar pistas más finas (nombre de sección exacta, tres palabras clave, párrafo aproximado) antes de saltar a plan (a).

- **Corrección directa de errores conceptuales de Tole**: dos veces en la sesión (elección `ChatClient` con justificación "más control", elección `ChatModel` genérico con justificación "por si cambiamos de modelo"). Ambas correcciones sin softening pero con explicación completa de la razón real. Bien recibidas. Refuerza el patrón: correcciones directas + andamio conceptual explicando el mecanismo real.

- **Confusión "Camino A/B" (bucle ReAct) vs "Opción A/B" (config `application.yml` de S18)**: al arrancar S19 Tole preguntó "era la opción B, no?" refiriéndose al camino ReAct. Aclarado que hay dos pares A/B distintos: config del yaml (resuelto en S18, ganó B empíricamente por el log del handshake) y camino del bucle (pendiente para S19, lockeado A). Buen momento para desambiguar antes de arrancar.

- **Cazado de imprecisión mía "la única implementación disponible"**: Tole cuestionó al vuelo esa frase mía sobre `ChatModel`. Corregido en caliente: **"única implementación autoconfigurada por Spring en ESTE proyecto concreto"**, no "única existente". Existen muchas implementaciones (`OpenAiChatModel`, `OllamaChatModel`, etc.). Buena caza de vaguedad, refuerza el hábito de Tole de no dejar pasar frases imprecisas.

- **Cazado de import `Message` equivocado**: Tole importó `Message` de Anthropic (`com.anthropic...`) en lugar de Spring AI (`org.springframework.ai.chat.messages.Message`). Cazado en el segundo intento tras no compilar. **Aprendizaje reforzado**: cuando IntelliJ ofrece varios imports para el mismo nombre, verificar el package antes de dar Enter. Este mismo aprendizaje se aplicó dos veces más en la sesión (import `@RequestBody` de Swagger, colisiones futuras esperables).

- **Predicciones antes de ejecutar**: mantenido el patrón varias veces. Predicción del `docker compose ps` (parcialmente correcta, con matiz sobre falta de healthcheck en `erp-app` — deuda #6). Predicción de "una sola iteración porque el LLM pide tools de golpe" (casi correcta, faltaba la llamada final del LLM para sintetizar; refinada a "1 iteración del `while`, 3 llamadas al LLM en total"). Predicción de la respuesta del `curl` (correcta en estructura).

- **Cierre elegido con perspectiva**: al alcanzar bucle funcional + commit + push, propuse cerrar S19 aquí y dejar `LlmLoggingAdvisor` para S20. Argumentos: 4h ya consumidas, dos bugs sutiles resueltos, core del ejercicio en verde. Añadir el advisor ahora sería alargar sin necesidad y arriesga fatiga. Tole no aceptó cerrar inmediatamente, pidió seguir hasta commit ("seguimos"). Fatiga controlada, decisión suya, respetada. Cierre efectivo con commit + push + notas.

- **Regla `git status` doble aplicada por primera vez en este proyecto**: `git status` antes de `git add` (verificar untracked files), después de `git add` (verificar staged coincide con lo esperado). Ambos limpios y exactos. Regla lockeada operacional en este proyecto también.

- **Bilingüe con separador `---` y español sin tildes/ñ en commit**: aplicado sin fricción. Formato replicado desde `erp-mcp-server` y `document-analyzer-ai`.

- **Sobre candidaturas**: mencionadas 3 negativas nuevas al arrancar la sesión (una en 30' — filtro ATS puro). Sin drama, tratadas como ruido de agosto. Consistente con el enfoque táctico lockeado (reevaluación primera semana de septiembre, no ahora).

- **Sobre proyecto v2 conversacional**: Tole expresó interés al descubrir la distinción intra-execution vs inter-execution memory. Aparcado explícitamente como **deuda #8** para después de cerrar v1 con guardrails + HITL. Buena disciplina de scope: no dispersar el proyecto actual, apuntar la idea para su momento.

- **Multi-LLM (OpenAI/Ollama)**: Tole preguntó si haremos esto también en `erp-purchasing-agent`. Consultado el roadmap: **NO en este proyecto**, aparece explícitamente en Proyecto 4 (Fase 5) con AI Gateway multi-modelo. Regla lockeada reforzada: **cada proyecto de portfolio se justifica por lo que enseña, no por lo que implementa. Añadir features "por completitud" diluye el mensaje**.

## Deuda técnica activa (informativa, no bloquea S20)

Sin cambios respecto al prompt de continuación S19, salvo añadido de deuda #8:

1. Bug `LlmLoggingAdvisor` × `MessageChatMemoryAdvisor` en `/chat` de `document-analyzer-ai`.
2. Defensa en profundidad para fences markdown en `AnalyzeController` y `AnalyzePdfController`.
3. `MULTI/EXEC` para atomicidad de `saveAll` en `RedisChatMemoryRepository`.
4. Mover TTL a `application.properties` (ChatMemory + `CvAnalysisCache`).
5. `document-analyzer-ai` sin modelo Anthropic explícito.
6. `erp-mcp-server` sin actuator ni `/actuator/health`.
7. README.md pendiente en `erp-purchasing-agent`. Se hará cuando el proyecto tenga guardrails + logging + al menos un demo command reproducible.
8. **[NUEVA] `erp-purchasing-agent` v2 conversacional**: evolucionar el agente a chatbot conversacional con memoria persistida en Redis (patrón `RedisChatMemoryRepository` reutilizado de `document-analyzer-ai`), sesión por `userId` o `conversationId`, escenario de negocio: gerente del ERP mantiene conversaciones multi-turn sobre reposiciones, aprobaciones, consultas de proveedores. Requiere v1 cerrada (bucle + guardrails + HITL). Timing estimado: post-Fase 2.2 completa.

## Próxima sesión

**Sesión 20 — `erp-purchasing-agent` parte 4: `LlmLoggingAdvisor` adaptado + observabilidad por iteración**.

Objetivo: adaptar el `LlmLoggingAdvisor` del `document-analyzer-ai` al `ReActAgent`. Ver iteración por iteración qué tools invoca el LLM, con qué argumentos, qué resultado devuelve la tool, y cuántos tokens consume acumulados.

**Orden propuesto** (a discutir al arrancar S20):

1. **Revisar el `LlmLoggingAdvisor` original** en `document-analyzer-ai` HEAD `8e03afb`. Entender qué hace, qué NO hace, qué asume del contexto.
2. **Decisión de arquitectura**: en camino A puro sin `ChatClient`, los "advisors" de Spring AI no aplican directamente (los advisors viven en la pipeline de `ChatClient`). Necesitamos otro mecanismo. Opciones:
   - **Logging manual dentro del `while`**: `log.info(...)` explícito en cada iteración. Simple y directo.
   - **Aspecto AOP**: decorar el método `chatModel.call()` con `@Around`. Elegante pero introduce Spring AOP.
   - **Wrapper del `ChatModel`**: envolver el bean inyectado con un decorator que logea. Patrón decorator clásico Java.
3. **Elegir opción, justificar, implementar**.
4. **Añadir contador de iteraciones + tokens acumulados** (`response.getMetadata().getUsage()`).
5. **Test de humo con el mismo prompt de reposición**: verificar que ahora vemos en logs las 2 iteraciones + 3 llamadas al LLM + tools invocadas + tokens totales.
6. **Segundo commit del proyecto**: `feat(agent): add per-iteration logging for ReAct loop observability` con cuerpo bilingüe.

**No haremos en S20**:
- Guardrails formales (`max_iterations`, budget, timeout) — se van a S21.
- Tests unitarios con `MockChatModel` — se van a S22.
- HITL — se van a S23.
- README.md profesional — se hará cuando el proyecto tenga guardrails + HITL cerrados.
- Micrometer / OpenTelemetry — Fase 5 del roadmap, mucho más adelante.

**Alternativas si Tole prefiere cambiar de eje ese día**:
- AWS Sesión 12-E (Route Tables privadas + RDS auto-start deadline 24-ago, chat separado, urgente).
- Sesión de re-lectura guiada (MapStruct como tema propuesto, chat separado).
- Deuda técnica del arco Redis: bug `LlmLoggingAdvisor` × `MessageChatMemoryAdvisor` en `/chat` (45-60 min).

Ver también `PromptContinuacion-S20-ReActAgent-parte4-2026-08-23.md` cuando se genere.
