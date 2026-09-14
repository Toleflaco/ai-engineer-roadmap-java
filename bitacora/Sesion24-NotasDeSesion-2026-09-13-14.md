# Notas de sesion S24 — `erp-purchasing-agent`, fix deuda 16 (`contextLoads` sin Docker) + diseno HITL en pizarra

**Fecha**: sesion hibrida partida en dos por imprevisto de Tole. Bloque 1 arrancado el domingo 13-sep 2026 a las 12:00 y cerrado a las ~13:15 (~1h15 efectivas). Bloque 2 arrancado el domingo a las ~13:15 y cortado a las ~13:40 por imprevisto (25 min); reanudado el lunes 14-sep 2026 a las 5:45 y cerrado a las ~6:30 (~45 min). Total efectivo: **~2h25** repartidas en dos dias. Energia alta en las tres tramas.

Se cerro el bloque 1 con un commit empujado a `origin/main` (`6a24c04`) que salda la deuda 16: `./mvnw test` pasa 9/9 verde sin Docker levantado, y sigue en verde con Docker. El bloque 2 fue diseno puro en pizarra — cero codigo — con cinco decisiones cerradas sobre HITL que dejan S25 y S26 preparadas: S25 refactor a `@ConfigurationProperties` como precondicion, S26+ implementacion de HITL sobre constructor limpio.

---

## Conceptos cerrados y verificados con Tole en S24

1. **Cascada exacta que rompe `contextLoads` sin Docker**. El bean mas profundo es `mcpSyncClients` (un `List<McpSyncClient>` creado por el factory method `mcpSyncClients()` en `McpClientAutoConfiguration.java:188`). Ese factory llama a `McpSyncClient.initialize()` (linea 190) que hace un handshake MCP sincrono al arrancar. Sin el `erp-mcp-server` escuchando, el `HttpClient` del JDK devuelve `ClosedChannelException` → `ConnectException` → reactor lo propaga como `RuntimeException: Client failed to initialize by explicit API call`. Desde ahi cascadea hacia arriba: `mcpSyncClients` → `mcpToolCallbacks` (`SyncMcpToolCallbackProvider`) → `toolCallbackResolver` → `toolCallingManager` → `anthropicChatModel` → `reActAgent`.

2. **Los cinco autoconfigs MCP estan matcheados por `@ConditionalOnProperty spring.ai.mcp.client.enabled=true`**. Confirmado leyendo el `CONDITIONS EVALUATION REPORT`: `McpClientAutoConfiguration`, `McpToolCallbackAutoConfiguration`, `SseHttpClientTransportAutoConfiguration`, `StreamableHttpHttpClientTransportAutoConfiguration`, `StdioTransportAutoConfiguration`. Estan activos porque el `application.yml` no fija la property y el default es `true` cuando el autoconfig esta en el classpath. Sobrescribirla a `false` los descarta a todos y corta la cascada en la capa de conexion.

3. **`@SpringBootTest` sin mas anotaciones NO carga "menos" que produccion**. Correccion a una intuicion inicial de Tole ("en los tests no tiene que arrancar todo el contexto"). El `@SpringBootTest` carga **el mismo** `ApplicationContext` que produccion: los mismos autoconfigs matchean, los mismos beans se construyen, la misma `application.yml` se lee. La unica diferencia entre "arranca en produccion con Docker" y "revienta en test sin Docker" es si el servidor MCP responde al handshake, no que beans se instancian.

4. **Opciones de fix ordenadas por capa donde cortan**. (a) capa de conexion — apagar el autoconfig MCP en tests; (b) capa de bean — dejar que se autoconfigure pero sustituirlo por un mock (por ejemplo con `@MockitoBean`); (c) capa de test — levantar Testcontainers con el `erp-mcp-server` real. Elegida (a) por ser la mas honesta: no reemplaza ningun bean, no ensucia produccion, y reusa el mecanismo que Spring ya expone (`enabled: false`). Descartadas (b) y (c) por complejidad innecesaria para un `contextLoads`.

5. **Property override vs perfil de test**. Dos formas de aplicar (a): (1) property `spring.ai.mcp.client.enabled=false` sin perfil, (2) perfil `test` con `application-test.yml`. Elegida (2). Justificacion: el perfil `test` deja un sitio identificado donde meter futuros overrides — Redis, otros stubs — cuando lleguen. Pequeno coste extra hoy, ganancia mecanica futura.

6. **`@ActiveProfiles("test")` vs `spring.profiles.active=test` en `application.properties` de test**. `@ActiveProfiles` explicita por clase (visible en el codigo, quien activa el perfil se decide test a test). `spring.profiles.active` en `src/test/resources/application.properties` activaria el perfil implicitamente para todos los tests del proyecto. Elegida la anotacion por explicita — futuros tests de integracion que quieran el contexto real con MCP levantado no ponen la anotacion y punto.

7. **`application.yml` / `application-test.yml` no compilan**. Aclaracion a Tole: son ficheros de recursos, Maven los copia tal cual al classpath en `process-resources` / `process-test-resources` sin pasar por `javac`. El criterio para elegir entre property override y perfil no es "compilar si/no" sino quien decide que tests corren con que perfil.

8. **`src/main/resources/` vs `src/test/resources/`**. `main` es classpath de produccion — cualquier `.yml` de perfil de test que se meta ahi acaba embebido en el jar. `test` es classpath de test unicamente. Cuando Tole se adelanto y creo `application-test.yml` en `main`, se movio a `test/resources/`. Sin drama, error comun de ubicacion.

9. **Persistencia de Redis (AOF/RDB) no resucita TTLs expirados**. Aclaracion a Tole al hablar del TTL para HITL: AOF y RDB persisten los datos frente a reinicios de Redis, pero no reviven claves cuyo TTL ya vencio. Si guardas con TTL=1h y pasan 61 minutos, la clave esta muerta, aunque Redis nunca se cayera. La persistencia protege el "estar caido 5 minutos", no el "TTL de 24h".

10. **HTTP 202 Accepted como codigo para "pausado esperando aprobacion"**. Del RFC 9110: "The 202 (Accepted) status code indicates that the request has been accepted for processing, but the processing has not been completed". Encaja exacto con HITL: el agente ha empezado, no ha fallado, pero esta esperando aprobacion humana para continuar. Los 3xx son redirecciones (le dicen al cliente "ve a otro sitio"), no encajan aqui. Con 200 (completo), 202 (pausado) y 422 (guardrail excedido), el `POST /agent/run` cubre los tres caminos sin ambiguedad.

11. **HTTP 410 Gone como codigo para "runId expirado"**. Semanticamente correcto cuando sabes que el recurso existio pero ya no. Se usa en el `POST /agent/run/{runId}/approve` cuando el TTL de Redis ha vencido. El cliente reacciona relanzando el prompt original desde cero. No hay efectos colaterales que deshacer, porque hasta la pausa el agente solo ha razonado y leido, no ha escrito en el ERP.

12. **`@ConfigurationProperties` es un binder yml → POJO, no un contenedor de colaboradores**. Aclaracion a Tole al clasificar los 9 parametros del constructor de `ReActAgent`: los colaboradores (`ChatModel`, `ToolCallingManager`, `Clock`, `List<McpSyncClient>`) son beans con comportamiento que Spring cablea via autoconfig o `@Bean`. `@ConfigurationProperties` solo puede leer valores primitivos, strings, listas, mapas y otros POJOs del yml y meterlos en un POJO tipado. Los cinco `@Value` actuales (`inputCostPerMillionTokens`, `outputCostPerMillionTokens`, `maxIterations`, `maxTokensBudget`, `maxDurationMs`) son los que entran; los colaboradores se quedan como inyeccion normal.

13. **Patron Strategy explicado en pizarra a raiz de la opcion `HumanApprover`**. Tole pregunto que era eso y como se hacia. Explicado con pizarra: una interfaz (`HumanApprover` con metodo `decide(List<ToolCall>) -> ApprovalDecision`), varias implementaciones (`AlwaysHumanApprover`, `WhitelistApprover`, `LlmJudgeApprover`), Spring cablea la que toque via `@Configuration` o `application.yml`. El `ReActAgent` no cambia — recibe **un** `HumanApprover` y lo usa. Contexto profesional que se aporto: el patron Strategy es tabla rasa en Java empresarial, se da por hecho en un backend con nueve anos como el de Tole. La parte "de AI engineer" del diseno de HITL no esta en usar Strategy — esta en (1) reconocer HITL como problema real de sistemas agenticos, (2) saber donde meter el corte en el bucle ReAct, (3) disenar el shape del estado persistido entre pausa y aprobacion, y (4) razonar sobre que tools requieren HITL. Ahi es donde brilla el AI engineer.

## Decisiones de diseno HITL cerradas en el bloque 2

Cinco decisiones cerradas en pizarra. Guardadas en `S24-diseno-HITL.md` en el repo de notas. Resumen ejecutivo:

**1. Que tools requieren HITL**. Requieren HITL: (a) tools de escritura con impacto de negocio y (b) tools de lectura de datos sensibles (margenes, nominas, costes internos). NO requieren HITL: lecturas de datos ordinarios (proveedores, productos) y escrituras sin impacto de negocio (logs internos, auditoria). Fuera de alcance del agente: cumplimiento RGPD. Se resuelve filtrando en origen en el `erp-mcp-server`, no metiendo aprobacion humana en el agente.

**2. Como se interrumpe el bucle ante HITL**. Retorno normal con `status` en `AgentRunResult` (opcion b). El bucle sale con `break` cuando el LLM propone una tool call que requiere HITL. `run()` construye un `AgentRunResult` con `status=PAUSED_FOR_APPROVAL` que lleva las tool calls pendientes y el estado a persistir. Descartadas: excepcion `HumanApprovalRequired` (HITL es flujo normal, no error) y `HumanApprover` inyectado con Strategy (sobreingenieria para v1 con una sola regla — refactor trivial cuando el criterio se bifurque). La deteccion "esta tool requiere HITL" vive en un `if` hardcodeado dentro del bucle, consultando la lista de tools sensibles.

**3. Donde vive el estado entre pausa y reanudacion**. Redis, con TTL de 24h que se refresca en cada aprobacion. Descartadas: estado en el cliente (payload grande viajando en HTTP, imposibilidad de confiar en el estado devuelto sin firmar el payload) y estado en memoria del proceso (se pierde en reinicios, no escala horizontalmente). Si el TTL expira, el `runId` es irrecuperable y el usuario debe relanzar el prompt original desde cero — no hay efectos colaterales que deshacer. Comportamiento del TTL: se guarda al pausar con clave = `runId` (UUID), se refresca a 24h en cada aprobacion/rechazo intermedio (la ventana mide tiempo desde la ultima interaccion humana, no vida absoluta del run).

**4. Endpoints v1**. Dos endpoints:
- `POST /agent/run` con tres codigos: `200 OK` (completado), `202 Accepted` (pausado, body con `runId` + `originalPrompt` + `pendingToolCalls`), `422 Unprocessable Content` (guardrail excedido).
- `POST /agent/run/{runId}/approve` sin body (aprobacion en bloque implicita) con tres codigos: `200 OK` (reanudado y completado), `202 Accepted` (reanudado y pausado otra vez en nueva tool call HITL), `410 Gone` (`runId` expirado o inexistente).
Descartados en v1: `POST .../reject` explicito (el TTL de 24h hace de rechazo por olvido), `GET /agent/run/{runId}` (el cliente descubre el estado al intentar aprobar), y aprobacion granular por `toolCallId` (v1 en bloque; el caso "aprobar A y rechazar B" se resuelve relanzando el prompt "solo A"). Todos anotados como mejoras futuras.

**5. Impacto en `ReActAgent` y orden de refactors**. Orden (B): `@ConfigurationProperties` primero en S25, HITL despues en S26+. Justificacion: sin HITL el constructor de 9 parametros esta bajo el umbral 10+ y la refactor seria YAGNI; HITL lo cruza (anade `RunStateRepository` y `HitlProperties`), asi que la refactor deja de ser YAGNI. La refactor es mecanica y aislable, HITL es logica nueva con Redis, serializacion y endpoints — mezclarlas complica el diagnostico. Trade-off aceptado: la refactor anade una sesion al plan. Reparto de parametros tras S25: colaboradores (`ChatModel`, `List<McpSyncClient>`, `ToolCallingManager`, `Clock`) + `LlmPricingProperties` (`inputPerMtok`, `outputPerMtok`) + `AgentGuardrailsProperties` (`maxIterations`, `maxTokensBudget`, `maxDurationMs`) = 6 parametros. Tras S26 anadiendo HITL: 5 colaboradores + 3 properties (nuevo `HitlProperties` con lista de tools sensibles) = 8 parametros. Comodamente bajo el umbral.

## Bugs / rugosidades diagnosticados en S24

1. **`application-test.yml` creado en `src/main/resources/` por adelanto de Tole**. Al preguntarle a Tole por el formato del fichero principal (`ls src/main/resources/`), aparecio ya un `application-test.yml` que Tole habia creado al saltarse el paso de decidir donde ubicarlo. Contenido correcto (`spring.ai.mcp.client.enabled: false`), ubicacion equivocada. Fix: `mkdir -p src/test/resources/ && mv src/main/resources/application-test.yml src/test/resources/`. Sin consecuencias porque no llego a empaquetarse en un jar.

2. **Commit del bloque 1 con `git commit -m "..."` pegado dentro del editor**. Tole ejecuto `git commit` a secas en vez del one-liner sugerido, se le abrio el editor, y pego el bloque entero incluyendo `git commit -m "..."` como si fuera texto del mensaje. Git lo acepto y el commit `3d9a888` quedo con un titulo que empezaba por `git commit -m "test(app): ..."`. Fix: `git commit --amend` para reescribir mensaje. Primer intento tambien salio mal (solo elimino la parte `git commit -m "` inicial, dejo el resto de comillas y `-m`s intermedios) → `d3cd095`. Segundo `--amend` limpio → `6a24c04`. Mensaje bilingue final correcto. Coste: unos 8 minutos de friccion de herramienta. Cero coste conceptual — Tole lo reconocio como friccion de tooling, no de conocimiento.

## Estado del proyecto `erp-purchasing-agent`

**Ubicacion**: `~/proyectos/erp-purchasing-agent/`.

**Estado git**: `main` sincronizado con `origin/main`. Working tree clean. Un commit del arco S24 empujado:

- `6a24c04`: `test(app): isolate contextLoads from external MCP server via test profile`.

**Estructura al cerrar S24** (delta vs S23-C):
```
* src/test/resources/application-test.yml (NUEVO)
  - Contenido:
      spring:
        ai:
          mcp:
            client:
              enabled: false

* src/test/java/dev/toleflaco/erp_purchasing_agent/ErpPurchasingAgentApplicationTests.java
  - Anadido @ActiveProfiles("test") a nivel de clase.
  - Anadido import org.springframework.test.context.ActiveProfiles.
```

Ficheros de codigo de produccion: **sin cambios**. El fix es puramente de test.

**Firma actual de `ReActAgent.run()`**: `public AgentRunResult run(String prompt)` (sin cambios desde S23-A).

**`AgentRunResult`**: sin cambios desde S23-A.
```java
public record AgentRunResult(
    String text,
    long iterations,
    long tokensTotal,
    long durationMs,
    double costUsd
) {}
```

**Verificacion tests al cierre**:
- `./mvnw test` **sin Docker levantado** → 9/9 verde.
- `./mvnw test` **con Docker levantado** (verificacion doble para no romper el caso feliz) → 9/9 verde.

## Estado de repos relacionados (no se tocaron en S24)

- **`~/proyectos/document-analyzer-ai/`**: HEAD `8e03afb`, sin cambios.
- **`~/proyectos/erp-mcp-server/`**: HEAD `ff35c28`, sin cambios. Docker levantado al final del bloque 1 para la segunda verificacion; no se toco codigo del servidor.
- **`~/proyectos/coditramuntana/discography/`**: HEAD `f06b0b4`, sin cambios. Tole informo al arrancar el domingo que Coditramuntana escribio el 12-sep confirmando respuesta "la semana que entra" (semana del 14-sep). Tole sigue sin esperar contratacion.

## Deuda tecnica activa (actualizada al cierre de S24)

1. Bug `LlmLoggingAdvisor` × `MessageChatMemoryAdvisor` en `/chat` de `document-analyzer-ai`.
2. Defensa en profundidad para fences markdown en `AnalyzeController` y `AnalyzePdfController`.
3. `MULTI/EXEC` para atomicidad de `saveAll` en `RedisChatMemoryRepository`.
4. Mover TTL a `application.properties` (ChatMemory + `CvAnalysisCache`).
5. `document-analyzer-ai` sin modelo Anthropic explicito.
6. `erp-mcp-server` sin actuator ni `/actuator/health`.
7. README.md pendiente en `erp-purchasing-agent`.
8. `erp-purchasing-agent` v2 conversacional: memoria persistida en Redis, sesion por `userId`. Requiere v1 cerrada.
9. Constructor de `ReActAgent` con 9 parametros. Umbral 10+ sin cruzar. HITL lo cruza (anade `RunStateRepository` + `HitlProperties`). **Refactor a `@ConfigurationProperties` planificada para S25 como precondicion de HITL** (decision 5 del bloque 2).
10. Warning `uses or overrides a deprecated API` en `ReActAgent.java` desde S19.
11. Warning Mockito self-attaching como Java agent. Fix: `-javaagent:...byte-buddy-agent.jar` en surefire.
12. `costUsd` es `double`. Migrar a `BigDecimal` si el negocio requiere precision decimal exacta.
13. No hay validacion no-null del `prompt` al entrar en `ReActAgent.run()`. Fix futuro: `Objects.requireNonNull` o `@NotBlank` en el `AgentRunRequest`.
14. ~~`import static org.mockito.Mockito.*` (wildcard) quedo tras S23-B.~~ **SALDADA en S23-C** (`c15b490`).
15. ~~Valores de tokens hardcodeados en el escenario 2 (`100/50` y `200/30`) sin significado narrativo.~~ **SALDADA PARCIALMENTE en S23-C** (`c15b490`).
16. ~~Test `ErpPurchasingAgentApplicationTests.contextLoads` requiere Docker levantado.~~ **SALDADA en S24** (`6a24c04`) via perfil `test` + `spring.ai.mcp.client.enabled: false`.
17. **NUEVA S24 (anticipada)**: `application-test.yml` tendra que apagar Redis cuando se anada la infra Redis (S26 con HITL). Mismo patron que la deuda 16 saldada: property override en el perfil `test`. No bloquea nada hoy — anotada para que no sorprenda en S26.
18. **NUEVA S24 (anticipada)**: Redis debe anadirse al `docker-compose` del `erp-purchasing-agent`. Precondicion para S26. Precedente en la casa: `document-analyzer-ai` ya usa Redis para `ChatMemory` y `CvAnalysisCache` — reutilizar patron de configuracion.

## Reglas lockeadas en S24 (nuevas)

Ver reglas de S19-S23C — siguen vigentes. Nuevas de S24:

- **Regla lockeada en S24 (corte en la capa mas baja que resuelve el problema)**: cuando un test falla por dependencia externa, elegir la solucion que corta lo mas cerca posible del origen del problema (capa de conexion) antes de subir a capas mas complejas (mock de bean, Testcontainers). El corte bajo es mas honesto: no reemplaza componentes, no ensucia produccion, y aprovecha mecanismos que el framework ya expone.

- **Regla lockeada en S24 (perfil `test` sobre property override suelto)**: cuando hay que sobrescribir propiedades solo en tests, preferir un perfil `test` con `application-test.yml` en `src/test/resources/` sobre poner las properties sueltas en `application.properties` de test. El perfil deja un sitio identificado donde iran los futuros overrides (Redis, otros stubs) y explicita en el codigo del test (`@ActiveProfiles("test")`) quien activa el perfil.

- **Regla lockeada en S24 (ficheros de perfil de test viven en `src/test/resources/`, nunca en `src/main/resources/`)**: un `application-<perfil>.yml` de test en `main/resources` acaba embebido en el jar de produccion. Aunque no rompa nada mientras nadie active ese perfil en produccion, es un despiste conceptual.

- **Regla lockeada en S24 (`@ConfigurationProperties` es para configuracion, no para colaboradores)**: solo entran valores tipados que vienen del yml (primitivos, strings, listas, mapas, POJOs). Los colaboradores (`ChatModel`, `Clock`, repositories, etc.) son beans con comportamiento y se quedan como inyeccion normal en el constructor. Aunque un colaborador se "configure" en el yml (modelo, api-key), su instanciacion la hace el autoconfig del framework o un `@Bean`, no un binder.

- **Regla lockeada en S24 (refactor mecanica antes de logica nueva cuando esta cruzara un umbral)**: si una feature nueva va a cruzar un umbral de complejidad conocido (parametros del constructor, tamano de un metodo, responsabilidades de una clase), hacer primero la refactor mecanica que baja del umbral. Escribir la feature nueva sobre la estructura limpia evita reescribir tests dos veces y aisla el diagnostico si algo falla.

## Estado candidaturas (informativo)

Al cierre de S24:

- **Coditramuntana**: confirmacion el 12-sep de que responden la semana del 14-sep. Tole no espera contratacion. Sin accion planificada.
- **Otras candidaturas**: sin cambios reportados.

## Sesion de re-lectura guiada (informativa)

Sigue pendiente. Prompt en `PromptArranque-ReLecturaGuiada.md`. Tema propuesto: MapStruct o Testing (segun energia). No es S25.
