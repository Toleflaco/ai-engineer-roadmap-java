# Sesión 20 — 2026-08-24 (ReAct agent parte 4, observabilidad por iteración con logging manual)

**Continuación del arco iniciado en S16-S17 (conceptual), S18 (bootstrap + handshake MCP), y S19 (bucle ReAct mínimo funcional). Fase 2.2 del roadmap AI Engineer — Proyecto 2: `erp-purchasing-agent`. Sesión de código real: diseño socrático completo de qué observar en un bucle ReAct, decisión arquitectónica lockeada L vs A vs D (Logging manual vs AOP vs Decorator), 4 líneas de log estructuradas escritas por iteración, dos helpers privados extraídos, test de humo empírico exitoso con traza completa del bucle visible por primera vez, segundo commit del proyecto pusheado a GitHub.**

Duración efectiva: ~4h, lunes por la tarde. Energía sostenida sin fatiga significativa hasta la parte final (parte-2 del sub-paso 4B costó más — dos ediciones consecutivas donde Tole borró líneas incorrectas por cansancio visual, corregidas en review). Sesión con densidad alta de decisiones socráticas sobre qué campos merecen ser observables, un bloque de cultura general sobre escalones de observabilidad (nivel 1 al 5) por curiosidad expresa de Tole, y cierre en verde con `curl` real mostrando trace estructurada del bucle: iteration start → llm response → tool result → llm response → agent run completed.

## Repo state al cerrar

- **`~/proyectos/erp-purchasing-agent/`**: HEAD `c614bcd`, working tree clean, pusheado a `origin/main`. Segundo commit del proyecto sobre el core. Dos archivos modificados (`ReActAgent.java` +66 líneas, `application.yml` +9 líneas).
- **`~/proyectos/erp-mcp-server/`**: HEAD `ff35c28`, working tree clean. Levantado durante la sesión (`docker compose up -d`), sigue arriba al cierre (opcional pararlo con `docker compose down`).
- **`~/proyectos/document-analyzer-ai/`**: HEAD `8e03afb`, working tree clean. No tocado — solo consultado como referencia conceptual (código del `LlmLoggingAdvisor` original, precios `llm.pricing.*`).

Contenedores: `erp-app` y `erp-postgres` levantados. Proceso Spring Boot del agente parado tras cierre de sesión con `Ctrl+C`.

## Commits generados en la sesión

- **`c614bcd`** — `feat(agent): add per-iteration logging for ReAct loop observability`
  - Dos archivos modificados: `ReActAgent.java`, `application.yml`.
  - 72 insertions, 3 deletions.
  - Mensaje bilingüe con separador `---`, cuerpo español sin tildes/ñ.
  - Redactado con `git commit -F - <<'EOF'` heredoc por longitud del mensaje.
  - Pusheado a `origin/main`.

## Conceptos cubiertos

### Diseño socrático de qué observar en un bucle ReAct

Diseño en pizarra antes de tocar el editor. Punto de partida: revisión del `LlmLoggingAdvisor` original en `document-analyzer-ai` como **referencia conceptual, no como código a copiar**. El advisor viejo captura `latency_ms + tokens_in + tokens_out + cost_usd` de una llamada aislada al LLM. Estructura correcta para un caso donde cada request es una única invocación al modelo.

**Diferencia estructural en un agente ReAct**: un `run()` puede consumir N iteraciones del `while`, cada una con una llamada al LLM que puede o no disparar tool calls. Importa no solo cada llamada individual sino **la traza del bucle completo**. Los campos "llamada aislada" del advisor viejo son insuficientes.

**Reformulación socrática**: en la iteración N del bucle, ¿cuál es el **delta** (información nueva respecto a la iteración N-1)? Tres ángulos:

1. Qué aparece nuevo en el input de la iteración N.
2. Qué genera el LLM como output de la iteración N.
3. Qué produce la ejecución de tools de la iteración N (si las hubo).

Loguear estos tres deltas por iteración da la traza completa del bucle sin duplicar información.

### Cuatro líneas de log — semántica clara y responsabilidades separadas

Diseño final lockeado:

| Línea | Pregunta que responde | Cuándo se emite |
|-------|----------------------|----------------|
| 1. `iteration start` | "Voy a empezar una vuelta del bucle. ¿En qué estado estoy?" | ANTES de cada `chatModel.call()` |
| 2. `llm response` | "El LLM acaba de responder. ¿Qué me contestó?" | DESPUÉS de cada `chatModel.call()` |
| 3. `tool result` | "Acabo de ejecutar las tools que pidió. ¿Qué me devolvieron?" | DESPUÉS de `executeToolCalls()`, una línea por cada tool |
| 4. `agent run completed` | "El bucle terminó. Resumen total." | Una vez, al salir del `while` |

Formato final:

```
"iteration start iteration={} messages_size={} tokens_accumulated={}"
"llm response iteration={} finish_reason={} tool_calls={} tokens_in={} tokens_out={} text={}"
"tool result iteration={} tool_name={} tool_call_id={} result={}"
"agent run completed iterations={} tokens_total={} duration_ms={} cost_usd={}"
```

**Asimetría intencionada de las líneas 2 y 3**: la línea 2 captura la **decisión del LLM** ("quiero llamar estas tools" o "aquí tienes la respuesta"). La línea 3 captura **la realidad del sistema** cuando ejecutas esas tools. Son los dos lados del mismo intercambio — no son intercambiables. Sin la 2, no ves qué pidió el LLM. Sin la 3, no ves qué devolvieron las tools. Se necesitan las dos.

**Regla lockeada**: **en un bucle ReAct, la observabilidad requiere separar la decisión del LLM (línea 2) del resultado de ejecutar esa decisión (línea 3). Son eventos distintos, en momentos distintos del ciclo, con causas distintas de fallo**.

### Decisión de arquitectura L / A / D — por qué logging manual gana

Tres opciones evaluadas antes de teclear:

- **Opción L (logging manual dentro del `while`)**: `log.debug(...)` explícito en el bucle. Cero dependencias nuevas.
- **Opción A (AOP con `@Around`)**: aspecto que intercepta `chatModel.call()`. Introduce `spring-boot-starter-aop`.
- **Opción D (Decorator del `ChatModel`)**: envolver el bean con `LoggingChatModel implements ChatModel`, `@Primary`.

**Problema estructural que descarta A y D**: nuestro logging necesita ver tres cosas por iteración (input del LLM, output del LLM, resultado de tools). Esas tres cosas ocurren en **dos beans distintos** (`ChatModel` y `ToolCallingManager`). Cualquier solución de interceptación obliga a interceptar ambos, o queda cojo.

En cambio, en el `while` del `ReActAgent` las tres cosas están **al alcance de la mano en la misma función**. Es el único sitio donde el logging está estructuralmente bien ubicado.

**Regla lockeada**: **cuando la observabilidad necesita ver eventos que ocurren en múltiples beans dentro de un mismo flujo lógico, el sitio correcto no es interceptar cada bean, sino loguear inline donde el flujo está coordinado. La sobreingeniería de AOP o Decorator solo tiene sentido cuando el mismo evento se repite en muchos flujos distintos**.

### `finish_reason` — por qué es crítico y no redundante

Tole propuso inicialmente omitir `finish_reason` del log de la línea 2 pensando que era redundante con `hasToolCalls()`. Corrección con evidencia:

- Caso A: `finish_reason = "end_turn"` — el LLM terminó su razonamiento y dio respuesta final. Todo bien.
- Caso B: `finish_reason = "max_tokens"` — el LLM se cortó a mitad de generación por límite de tokens. La respuesta está **truncada** y el bucle sale igual que en el caso A.

Ambos casos hacen `hasToolCalls() = false` (sales del bucle) por razones **completamente distintas**. Sin `finish_reason` en el log, no puedes distinguir "terminó" de "se cortó".

**Regla lockeada**: **`finish_reason` siempre se loguea. Es la única forma de distinguir un `end_turn` natural de un `max_tokens` truncado**. Ambos matan el bucle igual pero significan cosas radicalmente diferentes.

### Truncado de payloads largos — patrón para `text` y `tool result`

El texto generado por el LLM en la iteración final puede pasar de 1500 caracteres (tabla markdown gigante). El `responseData` de una tool como `listLowStockProducts` puede pasar de 1200 caracteres (JSON con N productos). Loguear entero satura los logs.

Patrón lockeado: **truncado a 200 caracteres con sufijo `...(truncated, N total chars)`**.

Ventajas: se ve el inicio real del payload (útil para "aha, la tool devolvió algo que empieza con `[{`, formato correcto"), y se sabe la longitud total (útil para "aha, la respuesta pesa 1738 chars, quizás excesiva"). Metadata pura sin contenido pierde la señal de "cómo empieza el payload".

Aplicado en dos sitios: `text` dentro del helper `logLlmResponse` (variante con `text` presente), y `responseData` dentro del `for` de tool results. Cálculo inline con operador ternario (no merece helper aparte, son 2 líneas repetidas).

**Regla lockeada**: **para payloads potencialmente largos, truncar mostrando inicio (200 chars) + longitud total. Ni entero (satura), ni solo metadata (pierde señal de forma)**.

### Formato de tool calls — helper `formatToolCalls` con `Collectors.joining`

El output del LLM incluye `List<AssistantMessage.ToolCall>`. Cada tool call tiene `.name()` y `.arguments()` (String JSON crudo). Necesitamos formato compacto para una sola línea de log.

Decisión Opción raw (loguear el JSON tal cual sin parsear) sobre Opción parseo (parsear con Jackson y reformatear). Razón: DEBUG es para debug, no para stakeholders. JSON compacto es perfectamente legible. Parseo añade excepciones, dependencia, líneas — YAGNI.

Formato: `[nombre({"arg":valor}), nombre({"arg":valor})]` o `[]` si vacío.

Implementación con stream + `Collectors.joining`:

```java
return toolCalls.stream()
    .map(tc -> tc.name() + "(" + tc.arguments() + ")")
    .collect(Collectors.joining(", ", "[", "]"));
```

**Curiosidad útil aprendida**: `Collectors.joining(separador, prefijo, sufijo)` devuelve `"[]"` automáticamente cuando el stream está vacío. El `if (toolCalls.isEmpty()) return "[]"` inicial que Tole escribió es técnicamente redundante y puede eliminarse.

### Extracción de helper — criterio de cuándo sí y cuándo no

Análisis previo antes de teclear: método `run()` iba a pasar de ~21 a ~40-45 líneas. Umbral personal de Tole para extraer helpers: ~35 líneas.

Candidatos evaluados:
- **Formateo de tool calls**: 5-8 líneas de lógica de string. Ensucia el bucle. **Extraído** a `formatToolCalls(...)`.
- **Truncado del result**: 3-4 líneas inline. No merece helper. **Inline**.
- **Cálculo de coste**: 1-2 líneas de aritmética. No merece helper. **Inline**.
- **Logging de la respuesta del LLM**: repetido en dos sitios (fuera del `while` y dentro). **Extraído** a `logLlmResponse(...)`.

**Decisión sobre `logLlmResponse`**: sin efectos secundarios sobre estado del bucle (no muta `iteration` ni acumuladores). Solo loguea. Los tokens se acumulan **inline** en los dos sitios donde se invoca el helper. Mezclar logueo con mutación de estado en un helper es "la clase de decisión que te muerde en refactors futuros".

**Regla lockeada**: **helper si (1) el bloque es >5 líneas y (2) se usa en 2+ sitios O contamina la legibilidad del contexto padre. Inline en el resto. Y crítico: si un helper muta estado, extraer solo el logueo y dejar la mutación inline**.

### Escalones de observabilidad en la industria (2026)

Bloque de cultura general por pregunta expresa de Tole ("¿esto es el modo habitual profesionalmente?"). Cinco niveles:

1. **`System.out.println`**: nivel spaguetti. No lo usa nadie profesional.
2. **SLF4J + formato clave=valor** (lo que hemos hecho hoy): mínimo profesional aceptable. `grep` como herramienta única.
3. **Logs estructurados JSON + correlation IDs + MDC**: JSON con campos indexables, `run_id` propagado por MDC a todas las líneas del thread. Backend en Elasticsearch/Loki/Splunk/Datadog. Aquí vive el 80% de proyectos serios.
4. **OpenTelemetry (traces + metrics + logs correlacionados)**: estándar de facto 2024-2026. Cada llamada al LLM es un span con atributos (`llm.tokens_in`, `llm.finish_reason`). Cada tool call es un span hijo. Backend Grafana Tempo/Jaeger/Honeycomb/Datadog APM.
5. **LLMOps específico (LangSmith, LangFuse, W&B Traces, Arize Phoenix)**: herramientas construidas para agentes. Replay, comparación de versiones de prompts, etiquetado para fine-tuning.

Posicionamiento del proyecto: **nivel 2 sólido**. Para aprendizaje es el sitio correcto. Los campos que hoy metemos en `log.debug("{}", var)` mañana los metemos en `span.setAttribute("llm.tokens_in", value)` — la sustancia es la misma, solo cambia el vehículo. Migración natural cuando el proyecto se despliegue.

**Vocabulario introducido**: MDC (Mapped Diagnostic Context, `ThreadLocal` de SLF4J), correlation IDs, OpenTelemetry, Grafana Loki (backend de logs, hermano espiritual de Prometheus/Tempo). Sin implementar nada — solo mapa mental para futuras fases del roadmap.

### Costes y precios de Claude Sonnet 4.5 — inyección con `@Value`

Precios en `application.yml`:

```yaml
llm:
  pricing:
    input-per-mtok: 3.0
    output-per-mtok: 15.0
```

Inyección en constructor con `@Value("${llm.pricing.input-per-mtok}")` y `@Value("${llm.pricing.output-per-mtok}")`. Patrón replicado del `LlmLoggingAdvisor` original en `document-analyzer-ai`.

**Convención de orden de parámetros del constructor**: beans inyectados por tipo primero (`ChatModel`, `List<McpSyncClient>`), `@Value` al final. Convención Spring — no obligatoria pero legible.

Fórmula del coste: `(promptTokens / 1_000_000.0) * inputCostPerMillionTokens + (completionTokens / 1_000_000.0) * outputCostPerMillionTokens`. Formateado con `String.format("%.6f", cost)` para evitar notación científica.

**Sobre incluir coste en cada iteración vs una vez al cierre**: decisión lockeada de **una sola vez al cierre** (línea 4) con acumulados. Loguear por iteración añade N líneas cuando el valor es acumulativo y se lee mejor al final. Cost inline crecería 4x el volumen del log sin aportar dato nuevo.

## ⭐⭐⭐ Frases para entrevista

- **Sobre observabilidad en bucles ReAct**:
  > "En un bucle ReAct, la observabilidad requiere separar la decisión del LLM del resultado de ejecutar esa decisión. Son eventos distintos, en momentos distintos del ciclo, con causas distintas de fallo. Si solo capturas uno, la mitad del diagnóstico se pierde."

- **Sobre por qué logging manual sobre AOP/Decorator**:
  > "Cuando la observabilidad necesita ver eventos que ocurren en múltiples beans dentro de un mismo flujo lógico, el sitio correcto no es interceptar cada bean, sino loguear inline donde el flujo está coordinado. La sobreingeniería de AOP o Decorator solo tiene sentido cuando el mismo evento se repite en muchos flujos distintos."

- **Sobre `finish_reason`**:
  > "`finish_reason` siempre se loguea. `end_turn` y `max_tokens` matan el bucle igual pero significan cosas radicalmente diferentes — respuesta completa vs respuesta truncada. Sin ese campo no puedes distinguirlos y arruinas cualquier diagnóstico posterior."

- **Sobre truncado de payloads**:
  > "Para payloads potencialmente largos, se trunca mostrando inicio y longitud total. Ni entero (satura logs), ni solo metadata (pierde la señal de forma). El inicio te dice cómo empieza el payload, la longitud te dice si el volumen es sospechoso."

- **Sobre extracción de helpers**:
  > "Extraigo a helper si el bloque supera 5 líneas y se usa en 2+ sitios, o si contamina la legibilidad del contexto padre. Y crítico: si un helper mutaría estado del método padre, extraigo solo la parte pura y dejo la mutación inline. Mezclar logueo con mutación de estado en un helper es la clase de decisión que te muerde en refactors."

- **Sobre roadmap de observabilidad**:
  > "Hoy tengo logs estructurados con SLF4J por simplicidad, pero el diseño de qué observar por iteración está pensado para migrarse a OpenTelemetry sin rediseñar. Cada campo del log de hoy se convierte en un atributo de span mañana — la sustancia es la misma, solo cambia el vehículo."

- **Sobre coste en agentes vs llamadas aisladas**:
  > "En una llamada aislada al LLM el coste es acotado y predecible. En un agente ReAct, un solo `run()` puede consumir 5, 10, 20 llamadas al LLM según la tarea. Loguear coste acumulado por request te da la baseline para guardrails futuros — sin ese dato, cualquier umbral de budget es arbitrario."

## Bugs diagnosticados empíricamente durante la sesión

### Bug 1 — `iteration++` prematuro en la línea `iteration start` del bucle

Al añadir la primera versión de la línea 1 dentro del `while`, Tole usó `iteration++` como argumento del log. Bug conceptual: `++` incrementa en cada log, y se me dijo explícitamente en el sub-paso 4A que `iteration` **no debía tocarse hasta 4B**. El incremento va al final del bucle, no dentro del argumento del log.

Cazado en review pre-compilación. Corregido a `iteration` sin post-incremento.

**Regla reforzada**: los sub-pasos del plan tienen orden por una razón. Adelantar cambios "aprovechando que estoy aquí" rompe el aislamiento de la prueba y contamina el diagnóstico si algo falla.

### Bug 2 — `totalPromptTokens++` como argumento del log

Mismo error conceptual que el bug 1. En la línea 1 del bucle, Tole usó `totalPromptTokens++` para loguear el acumulado. El `++` incrementa el acumulador en 1 cada iteración (arbitrariamente, sin relación con tokens reales). Los tokens se suman de verdad en 4B extrayendo `Usage`.

Cazado en review pre-compilación. Corregido a `totalPromptTokens + totalCompletionTokens`.

### Bug 3 — Format specifier `%d` sobre `double`

Al escribir la línea 4 (agent run completed), Tole usó `String.format("%d", cost)` donde `cost` es `double`. En 4A `cost` valía 0.0 por acumuladores vacíos, así que no explotaba visualmente. En runtime real con `cost` distinto de cero, `%d` sobre `double` lanza `IllegalFormatConversionException`.

Cazado en review antes de arrancar 4B. Corregido a `%.6f` (mismo specifier que el advisor viejo).

**Regla reforzada**: format specifiers de Java son estrictos por tipo. `%d` = enteros, `%f` = decimales. Un bug latente por escribir `%d` sobre `double` puede tardar en aparecer si el valor accidentalmente es 0 al probar.

### Bug 4 — Línea de log rota copiada como statement suelto

Durante 4B-parte-1, Tole insertó una línea `log.debug("llm response iteration={}...", );` fuera de contexto entre el `while` y el `double cost = ...`. Sin argumentos, con `,)` final sin variables. Aparentemente copy-paste accidental.

Al pedir el borrado, en la primera iteración de fix Tole borró **más** de la cuenta: eliminó también el `log.debug("agent run completed ...` original, dejando sus argumentos huérfanos flotando como statements sueltos que no compilaban.

Cazado en review de dos iteraciones. Fix guiado con "borra ESTO exactamente, escribe ESTO exactamente". Resuelto en la tercera iteración.

**Aprendizaje de proceso**: en la parte final de la sesión (después de ~3h), el cansancio visual reduce la precisión al editar código. Cuando hay que borrar y sustituir bloques, es más fiable pedir el bloque exacto de "borrar" y el bloque exacto de "escribir en su lugar" que descripciones abstractas. Correcciones más largas pero más precisas.

### Bug 5 — `log.debug` con 4 placeholders y solo 3 argumentos

Dentro del `for` de tool results, Tole calculó `resultToLog` con el ternario de truncado pero no lo pasó al `log.debug`. El mensaje tenía 4 `{}` (iteration, tool_name, tool_call_id, result) pero solo 3 argumentos (iteration, tool_name, tool_call_id). IntelliJ marca `resultToLog` como variable no usada.

Cazado en review. Corregido añadiendo `resultToLog` como cuarto argumento.

**Regla reforzada**: cuando IntelliJ subraya una variable como "unused" y el código compila, no ignorar el warning — típicamente indica que se calculó algo con la intención de usarlo y se olvidó el consumidor final.

## Estado de la app y roadmap

- **`erp-purchasing-agent`**: bucle ReAct con observabilidad por iteración funcional. 4 líneas de log estructuradas + 2 helpers privados. Verificado empíricamente con el mismo escenario canónico "reposición de stock" de S19: trace completa visible en logs (iteration start iter=0 → llm response iter=0 con tool_use → iteration start iter=0 con messages_size=3 → tool result iter=0 con truncado → llm response iter=0 con end_turn → agent run completed iterations=2 tokens_total=4922 duration_ms=8776 cost_usd=0.023034). Sin guardrails formales aún, sin tests unitarios, sin HITL. Segundo commit del proyecto en `origin/main`.
- **`erp-mcp-server`**: sin cambios (HEAD `ff35c28`). 13 tools disponibles.
- **`document-analyzer-ai`**: sin cambios (HEAD `8e03afb`).
- **Roadmap AI Engineer, Fase 2.2 — Proyecto 2 (`erp-purchasing-agent`)**: core ReAct + observabilidad resueltos. Siguientes hitos: guardrails formales (S21), tests unitarios con `MockChatModel` (S22), HITL (S23), README profesional al final.

## Correcciones y aprendizajes de proceso

- **Nueva regla lockeada por evidencia empírica de esta sesión**: en la parte final del ciclo cognitivo (después de ~3h de trabajo denso), la precisión al editar código cae medible. En dos ediciones consecutivas (bugs 4 y 5 de la lista) Tole borró líneas equivocadas o pegó líneas fuera de contexto. **Ajuste de método**: cuando la sesión pasa de las 3h y hay que hacer edición quirúrgica en bloques con dependencias, pasar de "borra la línea que sobra" a "borra EXACTAMENTE este bloque, escribe EXACTAMENTE este otro". Más letra pero menos rebotes.

- **Correcciones directas mantenidas sin softening**: cinco bugs cazados en review, corregidos con explicación completa del mecanismo. Ningún "sorry" ni "quizás me equivoco". Bien recibido por Tole — el patrón sigue funcionando.

- **Bloque de cultura general sobre escalones de observabilidad**: introducido por pregunta espontánea de Tole ("¿esto es el modo habitual profesionalmente?"). Respuesta con 5 niveles (println → SLF4J → JSON+MDC → OpenTelemetry → LLMOps específico). Sin implementar nada — solo mapa mental. Recibido bien, ayudó a contextualizar dónde está el proyecto y hacia dónde escalar. Vocabulario nuevo introducido con dosis correcta (MDC, correlation IDs, Loki, Grafana Tempo, LangFuse). Refuerza la nueva regla lockeada de S19 (una decisión por mensaje en terreno nuevo).

- **Broma sobre Loki (dios de Marvel) manejada bien**: Tole se disculpó por la broma, se le devolvió como cultura pop vs infra sin drama, seguido de aclaración técnica. Mantener el tono relajado cuando ayuda a bajar tensión sobre "no sé nada" es útil.

- **Autopercepción de Tole "me queda muchísimo por aprender"**: recibida con calibración honesta. El campo de observabilidad LLM tiene 2 años como disciplina — nadie es senior aquí todavía. Lo que Tole hizo en las 3h (diseñar 4 puntos de observabilidad estructurada desde primeros principios, cazar el crecimiento cuadrático del prompt solo, entender por qué `finish_reason` es crítico) es material de mid/senior en LLMOps, no de aprendiz. La sensación de "me falta" es correcta porque el campo es vasto, pero no es proporcional al ritmo real.

- **Tema candidaturas mencionado brevemente al arrancar**: "más negativas... no veo la luz, si te soy sincero". Manejado con validación breve (agosto es mes cabrón, ATS + humanos de vacaciones, reevaluación primera semana septiembre sigue en pie) sin insistir. Tole eligió seguir con técnica. Respetado. Al final de la sesión, autopercepción de aprendizaje fue el vehículo natural para volver a la calibración positiva sin forzar el tema laboral.

- **Momento "me lo dices tú todo... cuando encuentre trabajo espero que me ayudes igual"**: aclaración importante hecha en el momento. Tole seguía escribiendo el código concreto (nombres, sintaxis, orden), yo daba guías estructurales. Y compromiso explícito: seguiré ayudando cuando encuentre trabajo, no cambia nada. Tole aclaró "no lo decía a malas" — cerrado sin drama.

- **Regla `git status` doble aplicada**: antes y después de `git add`. Ambos limpios y exactos. Regla operacional lockeada del proyecto.

- **Commit con `git commit -F - <<'EOF' ... EOF`**: primera vez en este proyecto usando heredoc para commit mensaje largo con separador `---`. Alternativa al editor. Formato bilingüe replicado sin fricción, español sin tildes/ñ verificado.

- **Cierre de sesión propuesto en verde**: al confirmar `curl` funcional + commit + push, propuse cerrar S20 con generación de notas + prompt continuación. Tole aceptó explícitamente ambos ("hazlo tú").

## Deuda técnica activa (informativa, no bloquea S21)

Sin cambios respecto al cierre de S19, salvo que ahora **deuda #7 (README.md)** se acerca (falta guardrails + HITL para desbloquearlo):

1. Bug `LlmLoggingAdvisor` × `MessageChatMemoryAdvisor` en `/chat` de `document-analyzer-ai`.
2. Defensa en profundidad para fences markdown en `AnalyzeController` y `AnalyzePdfController`.
3. `MULTI/EXEC` para atomicidad de `saveAll` en `RedisChatMemoryRepository`.
4. Mover TTL a `application.properties` (ChatMemory + `CvAnalysisCache`).
5. `document-analyzer-ai` sin modelo Anthropic explícito.
6. `erp-mcp-server` sin actuator ni `/actuator/health`.
7. README.md pendiente en `erp-purchasing-agent`. Se hará cuando el proyecto tenga guardrails + HITL cerrados.
8. `erp-purchasing-agent` v2 conversacional: memoria persistida en Redis, sesión por `userId`. Requiere v1 cerrada.

**Nueva deuda potencial (no urgente)**: en Fase 5 del roadmap, migrar las 4 líneas de log actuales a spans OpenTelemetry con `run_id` en MDC. Diseño de campos ya pensado para eso — la migración será mecánica, no rediseño. Registrada solo mental, no en la lista formal (no aplica a v1).

## Próxima sesión

**Sesión 21 — `erp-purchasing-agent` parte 5: guardrails formales del bucle ReAct**.

Objetivo: proteger el `run()` de tres formas de descontrol: bucles infinitos (LLM que sigue pidiendo tools indefinidamente), consumo excesivo de tokens (budget), y latencia excesiva (timeout).

**Orden propuesto** (a discutir al arrancar S21):

1. **Definir los tres guardrails**: `max_iterations` (ej: 10), `max_tokens_budget` (ej: 20000), `max_duration_ms` (ej: 60000). Valores por defecto configurables en `application.yml` (`agent.guardrails.*`).
2. **Diseñar semántica de superación**: ¿lanzar excepción `GuardrailExceededException` desde dentro del `while`? ¿O devolver respuesta con estado `truncated=true`? Trade-off entre "fail-fast + observable" y "graceful degradation".
3. **Decisión de arquitectura**: los `if` de guardrail van **dentro del `while`**, después de acumular tokens de la iteración actual, antes de la siguiente `chatModel.call()`. Consecuencia natural del patrón loop-and-a-half.
4. **Loguear cuando un guardrail dispara**: nueva línea de log tipo `guardrail exceeded type={} value={} limit={}`. Extensión natural del patrón de 4 líneas ya establecido.
5. **Inyectar los límites**: `@Value` en el constructor, mismo patrón que los precios de S20.
6. **Test de humo con prompt normal**: verificar que no se dispara ningún guardrail en el escenario canónico.
7. **Test de humo con prompt patológico**: forzar disparo de guardrail (ej: `max_iterations=1` temporalmente para ver el fail-fast). Restaurar valor razonable después.
8. **Tercer commit del proyecto**: `feat(agent): add iteration, token budget and duration guardrails to ReAct loop` bilingüe.

**No haremos en S21**:
- Tests unitarios con `MockChatModel` — S22.
- HITL — S23.
- README.md profesional — cuando el proyecto tenga HITL cerrado.
- OpenTelemetry / spans — Fase 5.

**Alternativas si Tole prefiere cambiar de eje ese día**:
- AWS Sesión 12-E (Route Tables privadas + RDS auto-start, chat separado).
- Sesión de re-lectura guiada (MapStruct como tema propuesto, chat separado).
- Deuda técnica del arco Redis: bug `LlmLoggingAdvisor` × `MessageChatMemoryAdvisor` en `/chat` (45-60 min).

Ver también `PromptContinuacion-S21-Guardrails-2026-08-24.md`.
