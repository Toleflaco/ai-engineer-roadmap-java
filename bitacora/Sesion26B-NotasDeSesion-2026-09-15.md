# Notas de sesion S26-B — `erp-purchasing-agent`, diseno en pizarra del estado persistido de HITL + cierre de deuda 17 como obsoleta

**Fecha**: martes 15-sep 2026. Arrancada ~05:30 recien levantado con cafe. Cerrada tras la verificacion empirica del comportamiento Redis con Spring Boot 4.1. Sesion larga por la naturaleza del diseno en pizarra, con fatiga evidente en el tramo final ("esto se me esta haciendo dura", "no doy ni una" en un momento — sin razon empirica, ver rugosidad 2). Energia buena al inicio, decayendo en la ultima hora.

Sesion de diseno puro. **Sin commits nuevos, working tree clean al cierre**. Producto: (a) el diseno del estado persistido de HITL cerrado en pizarra — `AgentRunSession`, `AgentMessage`, `RunStateRepository`, `HitlProperties`, paquete `hitl/` plano — documentado en `S26-B-diseno-estado-persistido.md`. (b) Deuda 17 **cerrada como obsoleta** tras verificacion empirica: Spring Boot 4.1 + Spring Data Redis hace conexion Redis puramente lazy; el override en `application-test.yml` que la deuda 17 preveia no es necesario.

---

## Conceptos cerrados y verificados con Tole en S26-B

1. **`HitlProperties` en S26-B, no en S26-C**. Decision temprana. Argumento: `HitlProperties` es configuracion estatica (lista de tools sensibles + TTL), no estado por ejecucion. Solo dos campos. Patron `XxxProperties` ya lockeado en S25. Independiente en contenido del diseno del `AgentRunSession`. Coste marginal — se cierra en el mismo bloque.

2. **Distincion clave: `HitlProperties` (config estatica) vs `AgentRunSession` (estado por ejecucion)**. Tole partio confundiendo ambos — enumero "tokens de entrada, ultima respuesta del LLM, prompt inicial" como campos de `HitlProperties`. Recorrido Socratico corto: `@ConfigurationProperties` es lo que se lee del yml al arranque y no cambia entre ejecuciones; lo que cambia por ejecucion es `AgentRunSession`. Clarificado. Primera vez que Tole disena estado persistido con LLMs — la linea entre "config" y "estado" no es evidente hasta que se hace explicita.

3. **Campo `usefulDurationMs` en `AgentRunSession`**. Razonamiento sobre reanudacion: el TTL de 24h lo comprueba Redis solo (no hace falta `createdAt` en el objeto). Pero el guardrail `max-duration-ms` sigue vivo tras la pausa. Si `ReActAgent` recalcula `start = clock.instant()` al reanudar, el agente gana horas de trabajo util adicional (violacion silenciosa del guardrail configurado a 60 segundos). Solucion: guardar tiempo util acumulado antes de pausar. Al reanudar, `start = clock.instant()` marca solo el inicio del tramo nuevo, y el guardrail se comprueba como `usefulDurationMs + (now - start) > maxDurationMs`. Nombre y unidad siguen convencion del proyecto (`durationMs` en `AgentRunResult`).

4. **Camino 2 lockeado: DTO propio `AgentMessage` en vez de `List<Message>` de Spring AI**. Aplicada regla codigo real de S22: `./mvnw dependency:sources` + `unzip -p spring-ai-model-2.0.0-sources.jar org/springframework/ai/chat/messages/{Message,AbstractMessage,ToolResponseMessage}.java`. Hallazgos: `Message` es interfaz minima (solo `getMessageType()`), extiende `Content`. `AbstractMessage` tiene `Map<String, Object> metadata` con el propio `messageType` metido dentro como enum. `ToolResponseMessage` tiene constructor `protected` sin `@JsonCreator`, campos `final` sin setters, y un record anidado `ToolResponse(String id, String name, String responseData)` como contenido. Ninguna anotacion Jackson en ninguna parte. Consecuencia: serializacion polimorfica con `activateDefaultTyping` es invasiva y arrastra CVEs conocidos de Jackson generica. Camino 2 (record propio `AgentMessage`) es limpio con snake_case global ya puesto, Jackson maneja records desde 2.12 sin drama, coste es un mapper de ida y vuelta.

5. **Sin campo `pendingToolCalls` en `AgentRunSession`**. YAGNI aplicado tras razonamiento: al pausar, el `AssistantMessage` con las tool calls ya esta en `conversationHistory` (el bucle lo anade antes de detectar sensibilidad). Al reanudar, `ReActAgent` inspecciona el ultimo `AgentMessage` (`role=ASSISTANT` con `toolCalls` no vacio) y ejecuta desde ahi. No duplicamos informacion.

6. **Sin separacion `totalPromptTokens/totalCompletionTokens` en el `AgentRunSession`**. Discusion Socratica: la separacion solo se necesita para el calculo del coste (`LlmPricingProperties.inputPerMtok` y `outputPerMtok`). Pero el coste se acumula iteracion a iteracion — al reanudar, `costUsd` acumulado ya trae X + Y de las iteraciones previas, y la nueva iteracion calcula su tramo con los tokens del `ChatResponse` de ese momento. No hace falta historial de tokens desglosado. Basta con `totalTokens` (suma) y `costUsd` (suma). Si manana se quiere exponer prompt/completion por API (dashboard tipo FinOps), se revisita.

7. **`AgentRunSession` como record, `AgentMessage` como record, `HitlProperties` como record**. Consistente con precedentes del proyecto (`AgentRunResult`, `LlmPricingProperties`, `AgentGuardrailsProperties`). Jackson maneja records desde 2.12 con snake_case global sin anotaciones. Camino 2 elimina el impedimento que habria justificado clase.

8. **`runId` como `String` generado en `AgentController`, no en `ReActAgent`**. Contra-argumento presentado (testabilidad si `ReActAgent` lo genera con `IdGenerator` inyectable). Tole mantuvo su decision inicial. Razones que la sostienen: (a) observabilidad — log line con `runId` antes de arrancar el bucle, Fase 5 del roadmap; (b) `AgentRunResult.runId` siempre poblado, sin campo opcional segun se pause o no. El contra (llamadores futuros replicando la generacion) es especulativo — YAGNI, refactor barato cuando aparezca. Firma futura: `ReActAgent.run(String runId, String prompt)`. `String` en vez de `UUID` para no introducir un tipo nuevo en el proyecto — todos los IDs mueven como string.

9. **`Set<String> sensitiveTools` en `HitlProperties`**. Cambio desde `List<String>` propuesto inicialmente. Semantica correcta (conjunto sin duplicados, sin orden), performance despreciable con 13 tools pero mejor de todas formas. Spring Boot mapea `Set` desde YAML sin cambios adicionales.

10. **`Duration ttl` en `HitlProperties`**. Autodocumentada (no necesita convencion `XxxHours` en el nombre). Spring Boot mapea desde YAML aceptando notacion humana (`24h`, `30m`, `1500ms`).

11. **Prefix `agent.hitl` para `HitlProperties`**. Sigue el patron `dominio.subdominio` de los precedentes (`llm.pricing`, `agent.guardrails`). `hitl` es subdominio de `agent`. Descartado `hitl.values` (redundante — todas las properties son "values", no aporta informacion) y `hitl` a secas (rompe patron de dos niveles).

12. **Paquete `hitl/` como feature vertical plano**. Rechazadas: horizontal (dispersa el feature en 4 carpetas), y vertical con sub-paquetes (5-6 clases en 4 sub-paquetes = 1-2 por sub-paquete, indirection sin agrupacion). Razon para la vertical plana: IntelliJ mueve paquetes con refactoring seguro (F6), la estructura optima de hoy no tiene por que ser la de S26-E. Cinco clases planas dentro de `hitl/`. `AgentMessage` vive tambien en `hitl/`.

13. **`RunStateRepository` como clase concreta, no interfaz**. YAGNI aplicado tras contra-argumento. Los tests unitarios de `ReActAgent` mockean con Mockito, que mockea clases igual que interfaces. Extract Interface en IntelliJ es 30 segundos cuando aparezca una segunda implementacion real. Descartada Opcion 3 (Spring Data Redis con `@RedisHash`) por acoplar dominio con infraestructura y por la magia de Spring Data que no se controla facilmente.

14. **`AgentMessage.text` como `@Nullable`**. Replica la semantica de `AbstractMessage.textContent` de Spring AI, tambien `@Nullable`. Usa `org.jspecify.annotations.Nullable` (ya en el classpath a traves de Spring AI, sin dependencia nueva). Convencion en las listas: nunca null, `List.of()` si vacias.

15. **Descubrimiento empirico: Spring Boot 4.1 + Spring Data Redis es lazy**. Al anadir `spring-boot-starter-data-redis` al `pom.xml` (paso previo a la deuda 17), los 9 tests siguieron verdes sin Docker Redis. `./mvnw spring-boot:run` mostro las lineas de bootstrap de Spring Data Redis (`Bootstrapping Spring Data Redis repositories in DEFAULT mode`, `Found 0 Redis repository interfaces`) pero sin abrir socket. Sin repositorios `@RedisHash` ni codigo que use Redis en `contextLoads`, `RedisConnectionFactory` queda autoconfigurado pero dormido, sin validarse contra socket real. La hipotesis de partida de la deuda 17 (heredada de Spring Boot 3.x) no aplica. **Deuda 17 cerrada como obsoleta**, sin override en `application-test.yml`, sin commit. Dependencia revertida coherentemente con regla S26-A (no adelantar infra sin uso funcional).

## Rugosidades diagnosticadas en S26-B

1. **Confusion inicial `HitlProperties` vs `AgentRunSession`**. Tole enumero campos por ejecucion (tokens, respuesta LLM, prompt) como si fueran de `HitlProperties`. Nada raro — es la primera vez que disena estado persistido con LLMs, y la linea entre "config estatica" y "estado por ejecucion" no es evidente hasta que se hace explicita. Un recorrido corto lo aclaro. Anotado para no volver a asumir que la distincion es obvia.

2. **"No doy ni una" tras un contra-argumento debil sobre el prefix**. Momento de baja moral en la ultima hora, sin razon empirica: hasta ese momento Tole llevaba cerradas por si solo mas de 12 decisiones significativas del diseno (`usefulDurationMs`, sin `createdAt`, Camino 2, sin `pendingToolCalls`, sin separacion tokens, sin `text` en pausa, record, `@Nullable`, dos listas, paquete vertical, paquete plano, repository concreto). Respuesta correctiva: enumerar en el chat las decisiones cerradas. Tole confirmo el prefix `agent.hitl` en el mismo mensaje. Patron a vigilar: fatiga tardia + contra-argumento debil = riesgo de auto-descalificacion injustificada. Recap explicito es la respuesta correcta, no ceder al desafiante.

3. **Confusion con jerga interna "codigo real"**. Cuando Claude sugirio hacer una pausa "llevas hora y media de sesion densa con codigo real", Tole entendio "codigo Java escrito" (que era poco) en vez de "reglas de S22: `jar tf`, `unzip -p`, `-sources.jar`" (que era lo real). Diagnostico honesto de Tole en el turno: "es la primera vez que lo hago y me cuesta". Sin dano, aclarado. Nota metodologica: cuando el vocabulario interno del proyecto (jerga tipo "codigo real", "arco de commit", "guardrail") entra en un contexto de fatiga, conviene traducir a llano.

4. **`unzip -p` y `jar tf` con `while read` — Tole necesito la sintaxis exacta**. Iteracion Socratica correcta: propuso `jar tf spring-ai-2.0.0.jar | grep Message` (jar ficticio, Spring AI se distribuye en modulos separados) → Claude puntualizo que hay que hacer `dependency:build-classpath` primero → Tole compuso el pipe con `xargs -I{} jar tf {}` → Claude senalo que xargs sin `echo` de cabecera mezcla las salidas de varios jars sin decir cual → Tole reescribio con `while read jar; do echo "===== $jar ====="; jar tf "$jar" | grep Message; done`. Correcto. Sin drama. Anotado como iteracion valida — este patron es reutilizable en futuras inspecciones.

5. **Momento FinOps**. Al mencionar Claude "un dashboard de FinOps" como ejemplo hipotetico de por que separar prompt/completion tokens tendria sentido en el futuro, Tole pregunto que es FinOps. Aclarado en un mensaje corto (Financial Operations, disciplina de control de costes, herramientas Helicone/LangSmith/Vantage). Vocabulario nuevo asentado. Sin desvio de la sesion.

6. **`AssistantMessage$ToolCall` como nombre de clase nested confuso a primer golpe**. Tras el `jar tf` con el output, Tole pregunto que significa el `$` en `AssistantMessage$ToolCall`. Aclarado con ejemplo de codigo aproximado: es el separador de clases anidadas en Java compilado. Concepto asentado, con el matiz de por que importa para la decision Camino 1 vs Camino 2 (cuantas capas de tipos de Spring AI arrastras a Redis, mas superficie de anotaciones Jackson tienes que asegurar).

## Estado del proyecto `erp-purchasing-agent`

**Ubicacion**: `~/proyectos/erp-purchasing-agent/`.

**Estado git**: `main` sincronizado con `origin/main`. Working tree clean. **Cero commits nuevos en S26-B**. HEAD sigue en `4c8057d` (deuda 18 saldada en S26-A).

**Cambios in-memory durante S26-B, no commiteados**:
- Anadida temporalmente `spring-boot-starter-data-redis` al `pom.xml` para verificar comportamiento empirico → revertida al confirmar la Hipotesis A (Spring Boot 4.1 lazy Redis). Coherente con regla S26-A de no adelantar infra sin uso funcional inmediato.
- Fichero temporal `classpath.txt` generado por `dependency:build-classpath` y usado para inspeccionar los jars → borrado al cierre.

**Estructura al cierre**: identica a S26-A. Ni siquiera un stub en `hitl/`. La primera linea Java del feature HITL llega en S26-C.

**`ReActAgent`, `AgentController`, `AgentRunResult`**: sin cambios.

**Verificacion tests al cierre**: no se re-verifico al final tras revertir la dependencia. Estados conocidos verificados durante S26-B:
- Sin dependencia + Docker levantado: 9/9 (estado inicial, sin verificar formalmente al arrancar S26-B).
- Con dependencia + Docker levantado: 9/9 verde (verificado, output `Tests run: 9, Failures: 0, Errors: 0, Skipped: 0`).
- Con dependencia + Docker bajado: 9/9 verde (verificado — descubrimiento clave, ver concepto 15).
- Sin dependencia + Docker bajado: no se re-verifico. Sin cambios de codigo Java tras la reversion — se asume identico al inicio de S26-B, es decir, verde.

## Estado de repos relacionados (no se tocaron en S26-B)

- **`~/proyectos/document-analyzer-ai/`**: HEAD `8e03afb`, sin cambios. Ya no es referencia necesaria para el override de `application-test.yml` — la deuda 17 se cierra como obsoleta.
- **`~/proyectos/erp-mcp-server/`**: HEAD `ff35c28`, sin cambios. No se arranco en S26-B.
- **`~/proyectos/coditramuntana/discography/`**: HEAD `f06b0b4`, sin cambios.

## Deuda tecnica activa (actualizada al cierre de S26-B)

1. Bug `LlmLoggingAdvisor` × `MessageChatMemoryAdvisor` en `/chat` de `document-analyzer-ai`.
2. Defensa en profundidad para fences markdown en `AnalyzeController` y `AnalyzePdfController`.
3. `MULTI/EXEC` para atomicidad de `saveAll` en `RedisChatMemoryRepository`.
4. Mover TTL a `application.properties` (ChatMemory + `CvAnalysisCache`).
5. `document-analyzer-ai` sin modelo Anthropic explicito.
6. `erp-mcp-server` sin actuator ni `/actuator/health`.
7. README.md pendiente en `erp-purchasing-agent`.
8. `erp-purchasing-agent` v2 conversacional: memoria persistida en Redis, sesion por `userId`. Requiere v1 cerrada.
9. ~~Constructor de `ReActAgent` con 9 parametros~~. **SALDADA S25** (`67ce5b8`).
10. Warning `uses or overrides a deprecated API` en `ReActAgent.java` desde S19.
11. Warning Mockito self-attaching como Java agent. Fix: `-javaagent:...byte-buddy-agent.jar` en surefire.
12. `costUsd` es `double`. Migrar a `BigDecimal` si el negocio requiere precision decimal exacta.
13. No hay validacion no-null del `prompt` al entrar en `ReActAgent.run()`. Fix futuro: `Objects.requireNonNull` o `@NotBlank` en el `AgentRunRequest`.
14. ~~`import static org.mockito.Mockito.*` (wildcard)~~. **SALDADA S23-C** (`c15b490`).
15. ~~Valores de tokens hardcodeados en el escenario 2~~. **SALDADA PARCIALMENTE S23-C** (`c15b490`).
16. ~~Test `contextLoads` requiere Docker levantado~~. **SALDADA S24** (`6a24c04`).
17. ~~`application-test.yml` tendra que apagar Redis en S26 con HITL~~. **CERRADA COMO OBSOLETA S26-B**. Verificacion empirica confirmo que Spring Boot 4.1 + Spring Data Redis hace conexion lazy — sin `@RedisHash` ni codigo que use Redis, `contextLoads` no valida socket, la deuda no tiene remedio que aplicar. Si en S26-C+ aparece un test de integracion del `RunStateRepository` que si abre socket, se decidira Testcontainers vs mock vs embedded Redis en su momento — pero eso sera una deuda nueva, no la 17 reformulada.
18. ~~Redis debe anadirse al `docker-compose` del `erp-purchasing-agent`~~. **SALDADA S26-A** (`4c8057d`).

## Reglas lockeadas en S26-B (nuevas)

Ver reglas S19-S26-A — siguen vigentes. Nuevas de S26-B:

- **Regla lockeada en S26-B (verificar comportamiento empirico antes de aplicar remedios enunciados por hipotesis previas)**: extension de la regla de S26-A ("no cerrar deudas antes de que la dependencia funcional exista"). Cuando una deuda se enuncia asumiendo un comportamiento heredado de una version anterior de una libreria o framework, al llegar el momento de saldarla verificar primero que el comportamiento asumido sigue siendo el real en la version en uso. Aplicable a Spring Boot upgrades, pero tambien a Jackson, Mockito, Spring Security, etc. El remedio se aplica al problema real, no al enunciado. En S26-B se aplico al descubrir que Spring Boot 4.1 es lazy con Redis y la deuda 17 perdio su razon de existir.

- **Regla lockeada en S26-B (`XxxProperties` prefix sigue el patron `dominio.subdominio`)**: `llm.pricing`, `agent.guardrails`, `agent.hitl`. Cuando el subdominio es propio de un dominio existente, se anida (`agent.*`). Cuando es un dominio por si solo con parametros no anidables, se pone a nivel raiz (`llm.*`). El prefix `dominio.values` es antipatron — el "values" no aporta informacion, todas las properties son "values".

- **Regla lockeada en S26-B (fatiga tardia + contra-argumento debil = recap correctivo, no doblegarse al desafiante)**: cuando en la ultima hora de sesion Tole muestra baja moral ("no doy ni una") y el desafiante que Claude le acaba de presentar es debil (YAGNI puro, sin coste real inmediato), la respuesta correcta es enumerar en el chat las decisiones que Tole ha cerrado por si solo hasta ese momento, no seguir presionando con desafiantes. La intuicion cansada suele acertar en decisiones donde el contra-argumento es especulativo.

- **Regla lockeada en S26-B (paquete plano por feature con 5-6 clases; sub-paquetes cuando crezca)**: para un feature nuevo con 5-6 clases al inicio, paquete vertical plano (`hitl/`). Sub-paquetes solo cuando el conteo por sub-paquete rebase consistentemente 2-3 ficheros. Refactor con IntelliJ Move (F6) es 30 segundos.

- **Regla lockeada en S26-B (`unzip -p jar.jar path/inside.java` para leer sources sin extraer)**: extension de la regla codigo real de S22. `./mvnw dependency:sources` descarga los `-sources.jar`; `jar tf` lista contenido; `unzip -p` vuelca un fichero a stdout sin ensuciar el disco. Combinado con `dependency:build-classpath | tr ':' '\n' | grep libreria | while read jar; do echo "===== $jar ====="; jar tf "$jar" | grep TipoBuscado; done` para localizar en que jar vive un tipo cuando hay varios modulos.

- **Regla lockeada en S26-B (Camino 2 sobre Camino 1 cuando la libreria no coopera con Jackson por defecto)**: si el tipo externo que quieres serializar tiene constructores no-publicos, campos `final` sin setters, interfaz sin `@JsonTypeInfo`, o `Map<String, Object>` con enums metidos, mapea a un DTO propio del proyecto en vez de forzar la serializacion polimorfica de Jackson (`activateDefaultTyping` es invasivo y con CVEs conocidas). El coste es un mapper de ida y vuelta; el beneficio es control total sobre el shape del JSON persistido y cero anotaciones nuevas.

## Estado candidaturas (informativo)

Al cierre de S26-B:

- **Coditramuntana**: Tole no comento novedad al arrancar S26-B. Segun regla del prompt de arranque, ya no se pregunta activamente. Sin accion planificada.
- **Otras candidaturas**: sin cambios reportados.

## Sesion de re-lectura guiada (informativa)

Sigue pendiente. Prompt en `PromptArranque-ReLecturaGuiada.md`. Tema propuesto: MapStruct o Testing (segun energia). No es S26-C.

## Preparacion para S26-C

S26-C es la primera sesion de implementacion Java del feature HITL. Estado de preparacion al cierre de S26-B:

- **Diseno cerrado**: `S26-B-diseno-estado-persistido.md` guarda el shape final del `AgentRunSession`, `AgentMessage`, `RunStateRepository`, `HitlProperties`, y la estructura del paquete `hitl/`.
- **Precondiciones infra cerradas**: docker-compose con Redis (S26-A). Deuda 17 cerrada como obsoleta (S26-B) — no hay override que anadir.
- **`spring-boot-starter-data-redis` sin anadir en `pom.xml`**: coherente con la regla de S26-A. Se anade en S26-C cuando se codifique `RunStateRepository`.
- **Orden natural de S26-C**: (a) `HitlProperties` (record + entradas en `application.yml` con tools sensibles reales del catalogo del `erp-mcp-server`), (b) `AgentMessage` + `MessageMapper` con test unitario minimo del mapper, (c) `AgentRunSession` (record), (d) anadir `spring-boot-starter-data-redis` al `pom.xml`, (e) `RunStateRepository` con `RedisTemplate<String, String>` y `ObjectMapper` (implementacion + serializacion JSON del `AgentRunSession`). Ambicioso para una sola sesion — probablemente S26-C llegue hasta (b) o (c). Los cambios en `ReActAgent` (deteccion tool sensible, break del bucle, poblado de `AgentRunSession` en pausa, retoma) y `AgentController` (endpoint `/approve`, generacion de `runId`) llegan en S26-D o S26-E.
- **Cambios pendientes en `AgentRunResult` (para S26-D+)**: anadir campo `status` (enum: `COMPLETED`, `PAUSED_FOR_APPROVAL`, `GUARDRAIL_EXCEEDED`) y campos opcionales para el caso pausa (`runId`, `pendingToolCalls` extraidos del ultimo `AgentMessage`, `originalPrompt` extraido del primer `AgentMessage`). No en S26-C.
- **Endpoint nuevo pendiente (para S26-D+)**: `POST /agent/run/{runId}/approve` sin body. No en S26-C.
