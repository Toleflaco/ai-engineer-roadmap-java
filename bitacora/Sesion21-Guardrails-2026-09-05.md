# Sesion 21 — `erp-purchasing-agent`: guardrails formales del bucle ReAct

**Fecha**: sabado 5 de septiembre de 2026, manana (~9:30 - ~13:00, con parada de calentamiento largo).
**Duracion aproximada**: 3h 30min (incluyendo repaso previo del codigo y refresco de contexto).
**Estado inicial**: HEAD en `c614bcd`, working tree clean. Sin toques al codigo desde el 24-ago (10 dias por prueba tecnica Coditramuntana).
**Estado final**: HEAD en `91dfddf`, pusheado a `origin/main`. Cuarto commit del proyecto.
**Contexto de energia**: bajo al arrancar por parada larga y prueba tecnica intensa entre medias. Fue necesario un repaso completo del `run()` de S20 (mensaje detallado bloque por bloque) para reactivar el mapa mental antes de tomar decisiones de diseno. Una vez recuperado el hilo, energia sostenida hasta el cierre.

---

## Objetivo cumplido

Anadir tres guardrails configurables al bucle ReAct de `ReActAgent.run()` para prevenir descontrol:

- **`max-iterations`**: limite de llamadas totales al LLM (default 5).
- **`max-tokens-budget`**: limite de tokens acumulados prompt + completion (default 20000).
- **`max-duration-ms`**: timeout wall-clock desde el inicio de `run()` (default 60000).

Cuando cualquier guardrail se supera: `GuardrailExceededException` (RuntimeException) → `@ExceptionHandler` en `AgentController` → HTTP 422 + `ProblemDetail` RFC 7807 con campos custom (`guardrailType`, `value`, `limit`).

---

## Trabajo previo al codigo: repaso guiado del `run()` de S20

Antes de ninguna decision de diseno, sesion completa de repaso del `run()` heredado de S20 (~50 lineas del metodo, 2 helpers privados). El repaso se estructuro asi:

1. **Piezas del proyecto**: 4 clases (`ReActAgent`, `AgentController`, `AgentRunRequest`, `AgentRunResponse`).
2. **Constructor de `ReActAgent`**: 4 dependencias (`ChatModel`, `List<McpSyncClient>`, dos `@Value` de precios).
3. **`run()` bloque por bloque**:
   - **Bloque 1 (preparacion, una sola vez)**: mensajes, `toolCallbacks`, `toolCallingManager`, `AnthropicChatOptions`.
   - **Bloque 2 (contadores)**: `iteration`, `totalPromptTokens`, `totalCompletionTokens`, `start`.
   - **Bloque 3 (primera llamada, fuera del `while`)**: patron loop-and-a-half.
   - **Bloque 4 (bucle `while`)**: seis pasos por vuelta (`executeToolCalls` → log iteration start → extraer `ToolResponseMessage` → for de tool results → reconstruir `Prompt` → `chatModel.call()` → acumular tokens → `iteration++`).
   - **Bloque 5 (cierre)**: coste, log final, return.
4. **Los dos helpers**: `formatToolCalls` (cosmetico), `logLlmResponse` (solo loguea, sin efectos secundarios).
5. **Tres conceptos enterrados a refrescar**: statelessness de Anthropic, separacion `ChatModel` vs `ToolCallingManager`, `hasToolCalls()` como unica condicion de salida (y por tanto: **exactamente el problema que S21 resuelve**).

Termino la fase de repaso con el aluminio caliente: el mapa mental del bucle recuperado, listo para tomar decisiones sobre donde meter los checks.

---

## Seis decisiones de diseno cerradas (una por mensaje, metodo socratico puro)

### Decision 1 — Semantica de `max-iterations`

**Opciones**: (A) vueltas del `while`, (B) llamadas totales al LLM.

**Elegido**: **B (llamadas totales al LLM)**. Coherente con la semantica que ya se fijo en S20 en el log de cierre (`iteration + 1` precisamente para incluir la primera llamada fuera del `while`).

**Sub-decision B.1 vs B.2 (implementacion)**: reutilizar `iteration` cambiando su semantica (B.1) vs variable nueva `llmCalls` (B.2).
**Elegido**: **B.1**. Un solo contador, una semantica, se acaba el `+1` cosmetico. Menos superficie de bug.

### Decision 2 — Semantica de `max-tokens-budget`

**Opciones**: (A) suma total, (B) dos umbrales separados input/output, (C) presupuesto en dolares.

**Elegido**: **A (suma `totalPromptTokens + totalCompletionTokens`)**. Simple, un solo umbral, ya se acumulaba desde S20.

**Discusion adicional importante**: pregunta de Tole sobre si C se usa profesionalmente. Respuesta detallada:
- En produccion **no se usa C dentro del bucle del agente**.
- Se usa: (1) rate limits del proveedor en TPM/RPM, (2) presupuestos por request en tokens totales (equivalente a A), (3) presupuestos organizacionales en dolares en **otra capa** (dashboards FinOps, LangSmith, Helicone, LangFuse), (4) cuotas por tenant en Redis (en tokens, no dolares).
- **Regla mental**: el bucle protege recursos tecnicos (tokens, iteraciones, tiempo). El presupuesto en dinero es preocupacion de negocio, vive en otra capa. Cuando S22+ llegue a OpenTelemetry (Fase 5), estos campos pasan a ser atributos de span → dashboards → alertas de coste ahi fuera.

### Decision 3 — Semantica de `max-duration-ms`

**Opciones**: (A) wall-clock desde `start`, (B) solo tiempo de llamadas al LLM, (C) timeout individual por llamada.

**Elegido**: **A (wall-clock)**. Es la unica que responde "el usuario lleva X segundos esperando, ya basta". Ya tenemos `start = System.nanoTime()` desde S19.

### Decision 4 — Semantica al superarse un guardrail

**Opciones**: (E) excepcion fail-fast, (G) graceful degradation con `truncated=true`, (H) hibrida con `partialResponse` en la excepcion.

**Elegido**: **E (excepcion)**. Motivos:
- Fail-fast, un solo camino de exito, error handling centralizado.
- Es lo que harias por defecto en cualquier servicio Spring bien disenado.

**Observacion clave aportada por Claude que no estaba en las notas de Tole**: la respuesta parcial que "salvarias" con G a menudo no es texto util en un ReAct agent. Cuando revientas un guardrail a mitad, lo mas probable es que estes cortando en una vuelta donde el LLM estaba pidiendo tools (no dando la respuesta final). El texto util suele venir junto al `end_turn`, no antes. Eso debilita el argumento pro-G.

### Decision 5 — Forma de `GuardrailExceededException`

**Opciones**: (1) minima (solo message), (2) enum + type/value/limit, (3) enum + valores + contexto adicional.

**Elegido**: **2**. El punto dulce estandar. El JSON de error queda `{"type": "ITERATIONS", "value": 11, "limit": 10, "message": "..."}`. Cliente puede distinguir programaticamente. La 3 seria YAGNI.

**Sub-decision (enum interno vs separado)**: interno (`GuardrailExceededException.GuardrailType`). YAGNI aplicado. Si manana se necesita fuera, refactor de IntelliJ en 30 segundos.

### Decision 6 — Ubicacion de los checks en el bucle

**Opciones**: (A) al principio del `while`, (B) despues de reconstruir `currentPrompt` y antes de la proxima `chatModel.call()`, (C) al final del `while`.

**Elegido**: **B**. Los checks leen como "antes de gastar mas dinero en otra llamada al LLM, ¿me queda presupuesto?". Es la pregunta correcta.

**Detalle importante**: los checks solo protegen el interior del `while`. La primera llamada al LLM (fuera del `while`) no esta protegida. Aceptable porque: (1) es una sola llamada, no puede loopear; (2) el HTTP client de Spring AI ya tiene sus propios timeouts; (3) meter el check antes de la primera llamada no tiene sentido — los contadores estan todos a 0.

---

## Bugs cazados en review antes de ejecutar (patron consolidado de S19-S20)

Cinco correcciones aplicadas en pre-review antes de compilar, ninguna llego a runtime:

1. **`GuardrailExceededException` — campos sin `final` ni asignaciones**. Bug silencioso: los tres getters devolverian `null`/`0`. Fix: campos `final` (compilador exige asignacion) + `this.x = x` en el constructor.
2. **`GuardrailExceededException` — naming `GuardRailType`**. "Guardrail" es una sola palabra en ingles. Fix: `GuardrailType`, coherente con `GuardrailExceededException`.
3. **`GuardrailExceededException` — formato del mensaje con llaves y coma huerfana**. `"{type: %s, value: %d, limit: %d,"` sugiere JSON invalido. Fix: `"Guardrail exceeded: type=%s value=%d limit=%d"` estilo clave=valor coherente con S20.
4. **`GuardrailExceededException` — enum en medio de la clase**. Java convention: tipos anidados al principio o al final. Fix: enum al final.
5. **`GuardrailType` package-private**. Codigo fuera del package `exception/` no podria referenciarlo. Fix: `public enum GuardrailType`.

En el bloque de checks del bucle, dos correcciones adicionales:

6. **Parentesis huerfano en dos `log.debug`**. `log.debug("...", A, B, C));` — parentesis de mas. No compila.
7. **`nanoTime()` calculado dos veces en el bloque de duracion**. Ejecutar `System.nanoTime() - start` en el `if`, otra vez en el `log.debug`, otra en el `throw` da tres valores distintos. Cosmetico pero feo. Fix: extraer a `long elapsedMs` al principio del `if`.

Ademas, correccion aplicada por coherencia estetica:

8. **`totalPromptTokens + totalCompletionTokens` repetida tres veces en el bloque de tokens**. Suma determinista (no hay bug), pero mismo ruido visual. Fix: extraer a `long totalTokens`.

Y correccion propuesta rechazada correctamente por Tole con propuesta alternativa:

9. **Tole propuso inicializar `iteration = 1`** para "ahorrar" el `+1` cosmetico en los logs. Claude rechazo con tres consecuencias concretas: (a) log `iteration start` fuera del `while` mostraria `iteration=2` para la primera, (b) el check del guardrail con `iteration=1` inicial y `maxIterations=5` solo permitiria **4 llamadas** al LLM (bug silencioso), (c) el log de cierre necesitaria nuevo parche. **Regla mental**: cuando un contador tiene semantica clara ("cosas hechas"), no lo desplaces por conveniencia visual. Desplaza la vista en el log, no la fuente de verdad. Es el mismo motivo por el que se quito el `+1` cosmetico del log de cierre.

---

## Detalles tecnicos consolidados

### `application.yml` — bloque nuevo

```yaml
agent:
  guardrails:
    max-iterations: 5
    max-tokens-budget: 20000
    max-duration-ms: 60000
```

Bloque hermano de `llm.pricing.*`. Kebab-case. Linea en blanco de separacion respecto a los bloques adyacentes.

### `GuardrailExceededException` — forma final

- Package nuevo: `dev.toleflaco.erp_purchasing_agent.exception`.
- `extends RuntimeException`.
- Tres campos `final`: `type` (`GuardrailType`), `value` (`long`), `limit` (`long`). Tipo `long` unificado (cubre `int` de iteraciones/tokens y `long` de duracion).
- Constructor construye el mensaje via `String.format(...)` y lo pasa a `super(...)`.
- Enum `public GuardrailType { ITERATIONS, TOKENS, DURATION }` al final.
- Tres getters estandar.

### `ReActAgent` — cambios respecto a S20

- 3 campos nuevos `final`: `maxIterations`, `maxTokensBudget`, `maxDurationMs` (todos `long`).
- Constructor: 4 → 7 parametros (3 `@Value` nuevos).
- Import estatico: `import static ...GuardrailExceededException.GuardrailType.*;` para escribir `ITERATIONS` a pelo (satura menos visualmente).
- Import de la excepcion.
- `iteration` cambia semantica: **llamadas totales al LLM realizadas**. Se incrementa **justo despues** de cada `chatModel.call()` (dos sitios), no al final del `while`.
- Log `iteration start` usa `iteration + 1` (cosmetico, "arrancando llamada numero X").
- Log `agent run completed` usa `iteration` a secas (se quito el `+1` de S20).
- Log `tool result` sigue con `iteration` a secas: en ese punto vale "numero de la llamada al LLM que pidio estas tools" — semanticamente correcto.
- Bloque de checks entre reconstruccion de `currentPrompt` y siguiente `chatModel.call()`. Tres `if` en orden coherente con el YAML (iterations → tokens → duration). Cada `if` con su `log.debug` DEBUG antes del `throw`.

### `AgentController` — cambios respecto a S20

- Nuevo logger estatico + imports SLF4J.
- Import `HttpStatus`, `ProblemDetail`, `ExceptionHandler`.
- Metodo `handleGuardrailExceeded(GuardrailExceededException ex)`:
  - `@ExceptionHandler(GuardrailExceededException.class)`.
  - Log `WARN` con el mismo formato clave=valor (`guardrail exception handled type={} value={} limit={}`).
  - Construye `ProblemDetail.forStatusAndDetail(HttpStatus.UNPROCESSABLE_CONTENT, ex.getMessage())`.
  - `setTitle("Guardrail exceeded")`.
  - Tres `setProperty(...)` con los campos custom.
  - Retorna `ProblemDetail` directamente (Spring lo envuelve automaticamente).

**Nota tecnica sobre `UNPROCESSABLE_CONTENT`**: en Spring 6.2+/Java 17+ renombraron `UNPROCESSABLE_ENTITY` a `UNPROCESSABLE_CONTENT` para alinear con RFC 9110 (2022). Valor numerico sigue siendo 422. Nombre del reason phrase HTTP sigue siendo "Unprocessable Entity". `UNPROCESSABLE_ENTITY` sigue existiendo como alias deprecado.

### Estrategia de logging dual (decidida en S21)

- **DEBUG en `run()` antes del throw**: observabilidad del evento tecnico. Coherente con las 4 lineas de S20.
- **WARN en el handler despues del catch**: observabilidad del evento HTTP (cliente recibio 422). Util para dashboards y alertas futuras.
- Separa dos preguntas: ¿se disparo el guardrail? (DEBUG) vs ¿el cliente recibio un error? (WARN).

---

## Tests de humo empiricos

### Test normal (con `max-iterations: 5`)

Prompt canonico: "Revisa el stock de todos los productos y crea ordenes de compra para los que esten por debajo del minimo".

**Resultado**:
- HTTP 200 con `AgentRunResponse` valido.
- **5 iteraciones** consumidas (limite justo, al pelo).
- Ninguna linea `guardrail exceeded` ni `guardrail exception handled`.
- `agent run completed iterations=5 tokens_total=17321 duration_ms=18691 cost_usd=0.070071`.

**Observacion importante**: la tarea real consumio **exactamente el limite**. Si el LLM hubiera pedido tools en la iteracion 5 en vez de dar respuesta final (perfectamente posible), el guardrail habria disparado con tarea a medias. Nota mental: 5 es defensivo pero apretado para tareas complejas reales. En despliegue serio probablemente 10-15. Para v1 con proposito de "ver el sistema cerca del borde", 5 esta bien.

### Test patologico (con `max-iterations: 1` temporalmente)

Mismo prompt canonico.

**Resultado**:
- **HTTP 422** con `Content-Type: application/problem+json` (RFC 7807 aplicado automaticamente por Spring al detectar `ProblemDetail` como body).
- Cuerpo JSON: `{"detail":"Guardrail exceeded: type=ITERATIONS value=1 limit=1","instance":"/agent/run","status":422,"title":"Guardrail exceeded","guardrailType":"ITERATIONS","value":1,"limit":1}`.
- **Campo `instance`** con el path del request: autocompletado por Spring, util para debugging con multiples endpoints.
- Log DEBUG `guardrail exceeded type=ITERATIONS value=1 limit=1` desde `run()`.
- Log WARN `guardrail exception handled type=ITERATIONS value=1 limit=1` desde el handler.
- **La primera tool si se ejecuto** (`listLowStockProducts` visible en `tool result`). El guardrail corto antes de la segunda llamada al LLM, no antes de la primera ejecucion de tool. Coherente con ubicacion B.

Valor de `max-iterations` restaurado a `5` antes del commit.

---

## Reglas y patrones lockeados/aplicados

### Aplicadas de sesiones previas

- **Metodo socratico puro (S19)**: una decision por mensaje en terreno nuevo. Seis decisiones cerradas una a una, sin agrupar.
- **Predice antes de teclear**: aplicado en cada bloque (constructor, checks, log, handler). Capturo bugs 1-8.
- **`git status` doble antes y despues de `git add`**: aplicado sin fricciones.
- **Commits bilingues con `---`, sin tildes/n en la parte espanola**: mensaje del commit `91dfddf` cumple.
- **YAGNI**: enum interno vs separado, log en handler (Tole opto por WARN, decision suya), forma minima de la excepcion, no anadir `@ConfigurationProperties` para 7 parametros aun.
- **Regla "arranca sin error ≠ funciona"**: test patologico con `max-iterations=1` como verificacion real.

### Nuevas o refinadas en S21

- **Regla mental sobre contadores con semantica clara**: no desplaces la fuente de verdad por conveniencia visual del log. La cosmetica se hace en el `log.debug(...)`, no cambiando el punto de arranque del contador. Bug potencial cazado en decision 9 de la lista de bugs.
- **Regla sobre presupuesto en dinero**: el bucle protege recursos tecnicos (tokens/iteraciones/tiempo). El presupuesto en dolares vive en otra capa (dashboards, LLMOps, cuotas por tenant). No meter `max-cost-usd` en el bucle.
- **Estrategia de logging dual (DEBUG + WARN)**: separar evento de dominio del evento HTTP con niveles distintos. Prepara para alertas futuras sin refactor.

---

## Estado del proyecto al cerrar S21

- **Ubicacion**: `~/proyectos/erp-purchasing-agent/`.
- **Estado git**: HEAD `91dfddf`, working tree clean, pusheado a `origin/main`.
- **Cuatro commits totales**:
  - `852b592` — bootstrap (S18).
  - `0f192af` — ReAct loop minimo (S19).
  - `c614bcd` — per-iteration logging (S20).
  - `91dfddf` — guardrails (S21). 4 files changed.
- **Estructura de packages**:
  - `agent/ReActAgent.java` (~65 lineas de `run()`, sigue en umbral aceptable).
  - `controller/AgentController.java` (endpoint + `@ExceptionHandler`).
  - `dto/AgentRunRequest.java`, `AgentRunResponse.java`.
  - `exception/GuardrailExceededException.java` (nuevo).
- **`application.yml`**: `spring.*`, `server.port`, `llm.pricing.*`, **`agent.guardrails.*`** (nuevo), `logging.level.*`.

---

## Que viene en la proxima sesion (S22)

**Objetivo principal**: tests unitarios del `ReActAgent` con `MockChatModel`.

Contexto: hasta ahora toda la verificacion ha sido con smoke tests empiricos (`curl` real contra API real). Necesitamos:
- Tests deterministas que no consuman tokens de Anthropic.
- Verificar el comportamiento del bucle en escenarios controlados: respuesta directa sin tools, N iteraciones de tools, disparo de cada guardrail, etc.
- Cobertura del `AgentController.handleGuardrailExceeded` (opcional en S22, tambien puede ir a un test de integracion posterior).

**Decisiones abiertas para S22**:
- ¿Que framework de mocking? Spring AI trae utilidades para simular `ChatModel`. Investigar si hay algo tipo `TestChatModel` oficial, o si hay que construir uno manualmente con Mockito. Preferencia previa: usar lo oficial si existe (menos frigil), Mockito si no.
- ¿Que estructura de tests? Un test por caso (respuesta directa, N iteraciones, cada guardrail) vs suite parametrizada. Probablemente un test por caso para claridad, es el primer contacto con testing en este proyecto.
- ¿Como se maneja `List<McpSyncClient>` en test? Probablemente lista vacia + mock del `ToolCallingManager`.

**No hacer en S22**:
- No HITL (S23).
- No README (cuando cierre HITL).
- No OpenTelemetry (Fase 5).

---

## Deuda tecnica activa (informativa, sin cambios respecto a S20)

1. Bug `LlmLoggingAdvisor` × `MessageChatMemoryAdvisor` en `/chat` de `document-analyzer-ai`.
2. Defensa en profundidad para fences markdown en `AnalyzeController` y `AnalyzePdfController`.
3. `MULTI/EXEC` para atomicidad de `saveAll` en `RedisChatMemoryRepository`.
4. Mover TTL a `application.properties` (ChatMemory + `CvAnalysisCache`).
5. `document-analyzer-ai` sin modelo Anthropic explicito.
6. `erp-mcp-server` sin actuator ni `/actuator/health`.
7. README.md pendiente en `erp-purchasing-agent`. Se hara cuando el proyecto tenga HITL cerrado.
8. `erp-purchasing-agent` v2 conversacional: memoria persistida en Redis, sesion por `userId`. Requiere v1 cerrada.
9. **NUEVA S21**: constructor de `ReActAgent` con 7 parametros. En el limite de lo comodo. Si sube a 10+ en el futuro (probablemente cuando se metan mas guardrails o retries), refactor a `@ConfigurationProperties` para agrupar en POJOs (`AgentGuardrailsProperties`, `LlmPricingProperties`).
10. **NUEVA S21**: warning `uses or overrides a deprecated API` en `ReActAgent.java` desde S19. No es nuevo de S21 pero sigue sin investigarse. Investigar con `./mvnw compile -X` o `-Xlint:deprecation` cuando toque cierre de proyecto.

---

## Notas de proceso (meta)

- **Repaso guiado al retomar tras 10 dias sin tocar codigo funciona muy bien**. Antes del repaso: "no me acuerdo de nada". Despues del repaso: "ya recuerdo, retomamos ya". La inversion de tiempo (~30-40 min de repaso guiado bloque por bloque) se recupera con creces en la fluidez de las decisiones posteriores. Confirmar patron para futuros retornos tras pausas largas.
- **La correccion 9 (Tole propuso `iteration = 1`) muestra el metodo socratico funcionando**. Tole propuso mal, Claude explico las tres consecuencias concretas, Tole entendio y aplico correctamente. Sin regalar la respuesta, con dialogo. Sin sobre-explicaciones tampoco.
- **La densidad de decisiones de diseno de S21 (seis) sugiere que la fase de "guardrails" era mas rica de lo que parecia**. Anticipar en futuras sesiones que las tareas aparentemente sencillas ("anadir tres if") pueden esconder 5-6 decisiones cerradas con socratico.
- **Excepcion + `ProblemDetail`** fue conocimiento fresco de la prueba tecnica de Coditramuntana (commit `f06b0b4` con ADR-003). Coherencia arquitectonica entre proyectos: mismo patron de error handling en ambos.
