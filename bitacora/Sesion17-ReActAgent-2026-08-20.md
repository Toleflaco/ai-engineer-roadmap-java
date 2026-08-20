# Sesión 17 — 2026-08-20 (ReAct agent parte 1, bootstrap y conceptual)

**Arranque del Proyecto 2 del roadmap AI Engineer (Fase 2.2): `erp-purchasing-agent` como cliente ReAct del `erp-mcp-server` ya construido en S15.**

Duración efectiva: ~2h, jueves tarde. Café cargado al arrancar. Energía media, sostenida durante toda la sesión. Al arrancar Tole reportó 4 negativas ATS más (~16 acumuladas en 10 días); acordamos archivar sin drama y seguir con lo técnico. La reevaluación táctica de candidaturas sigue programada para primera semana de septiembre.

Sesión **muy conceptual, poco código**. Balance deliberado: primera sesión de un tema nuevo donde asentar bien el modelo mental evita retrabajo posterior. **Cero commits, cero código propio escrito**. Solo scaffolding vía Spring Initializr web. Es lo esperado y correcto en una S1 de nueva arquitectura.

## Repo state al cerrar

- **`~/proyectos/erp-purchasing-agent/`**: proyecto scaffoldado con Spring Initializr, **no es repo git todavía** (sin `.git/`). Se hará `git init` cuando haya algo mínimo funcional, no antes.
- **`~/proyectos/erp-mcp-server/`**: HEAD `ff35c28`, working tree clean. No tocado. Contenedores probablemente parados.
- **`~/proyectos/document-analyzer-ai/`**: HEAD `8e03afb`, working tree clean. No tocado.

Contenedores Docker probablemente todos parados. No se levantó ninguno durante la sesión.

## Commits generados en la sesión

**Ninguno**. Cero código propio escrito. Solo scaffolding descargado de Spring Initializr. Reconocido explícitamente y aceptado por Tole como cierre honesto de una S1 conceptual.

## Conceptos cubiertos

### ReAct pattern — el bucle NO itera por elementos de negocio

Confusión inicial de Tole: describió el bucle como un `while (haya productos bajos)` con lógica imperativa por elementos. **Modelo mental incorrecto** y hubo que reformular.

**Modelo correcto asentado**:
- El bucle ReAct itera por **decisiones del LLM**, no por elementos de datos.
- Cada vuelta del `while` = **1 llamada al LLM**.
- En cada llamada el LLM devuelve una de dos cosas:
  1. `tool_use { name, args }` → tu código ejecuta la tool, añade `tool_result` al historial, siguiente vuelta.
  2. Texto plano de respuesta final → sales del bucle.
- Condición de salida: NO es de negocio ("mientras haya productos"). Es "mientras el LLM siga pidiendo tools".

**Pseudocódigo mínimo** (4 líneas de lógica real, sin guardrails):

```
conversation = [system_prompt, user_message]
while (true):
    response = llm.call(conversation)
    if response contains tool_use:
        result = execute_tool(response.tool_name, response.args)
        conversation.append(response)              // el tool_use del asistente
        conversation.append(tool_result(result))    // el resultado
    else:
        return response.text                        // respuesta final
```

### Roles de mensajes en la API de Anthropic

Confusión importante de Tole: "yo creía que existía el system mi código, el LLM y las tools en el MCP Server". Le sonaba como si "user" fuera un actor separado.

**Aclarado**: los roles `system`, `user`, `assistant`, `tool_result` son **etiquetas convencionales** que **TU código Java pone a los mensajes** que envía al LLM. NO son actores. Cuando el usuario final teclea "necesito reponer stock" en un endpoint REST, tu controlador recibe ese string y lo mete en un objeto `Message(role="user", content="...")`. El LLM solo VE mensajes etiquetados; no ve tu código Java ni al usuario.

### Arquitectura de 3 actores desambiguada

Distinción crítica que Tole preguntó explícitamente: agente vs cliente MCP.

- **Agente**: lógica del bucle ReAct. Código Java de este proyecto (`erp-purchasing-agent`).
- **Cliente MCP**: librería `spring-ai-starter-mcp-client`. Habla protocolo MCP. Sabe hacer `tools/list` y `tools/call`. No sabe nada de LLMs ni de ReAct.
- **Servidor MCP**: el `erp-mcp-server`, con sus 13 tools anotadas `@Tool`.

El agente **usa** al cliente MCP. Ambos viven en el mismo JAR pero son **capas conceptuales distintas**. Si mañana se cambia el pattern (por ejemplo, plan-then-execute), toca el agente, no el cliente MCP. Si el protocolo MCP evoluciona, se actualiza la librería, no el bucle.

### Cómo descubre el LLM las tools disponibles

Tole predijo el flujo correctamente: (a) mi código le dice al LLM las tools, (b) en cada llamada le da el listado, (c) habrá una operación de listar tools en el servidor.

**Flujo confirmado**:
1. Al arrancar la app, el cliente MCP se conecta al servidor MCP e invoca la operación estándar `tools/list`. El servidor responde con el catálogo completo: nombres, `description` de `@Tool`, y JSON schema de parámetros con `description` de `@ToolParam`. El cliente lo guarda en memoria.
2. En **cada llamada al LLM**, el código Java construye el body HTTP con tres bloques principales:
   ```
   { "system": "...", "messages": [...], "tools": [...catálogo completo...] }
   ```
3. El bloque `tools` va en **cada** request porque la API de Anthropic es **stateless**: el LLM no recuerda nada entre llamadas.
4. Cuando el LLM responde `tool_use { name, args }`, el código Java delega en el cliente MCP con la operación estándar `tools/call`. El cliente hace la RPC al servidor MCP, ejecuta el método Java anotado, devuelve el resultado. El código lo mete como `tool_result` en el historial.

### Las `description` de `@Tool` son prompts, no javadoc

Punto crítico asentado con ejemplos reales del `erp-mcp-server`:

```java
@Tool(description = "List all products that have reached or fallen below their minimum stock threshold. ...")
@Tool(description = "Create a new supplier using a valid name and email address.")
```

Estos strings los **lee el LLM literalmente** para decidir cuándo llamar cada tool y cómo. Si en su día se hubieran escrito como `@Tool(description = "lowStockList")` (mentalidad javadoc), el LLM no sabría cuándo invocarla. **La calidad de estas descriptions es factor de primer orden** en si el agente funciona o alucina. Aplicable cuando Tole añada tools nuevas: escribir el `description` pensando "qué necesita leer un LLM que no conoce mi dominio".

### RPC = Remote Procedure Call

Tole preguntó. Aclarado con family tree histórico:
- **RPC**: invocar método/función que vive en otro proceso como si fuera local.
- Familia: RMI (Java 90s), CORBA, SOAP, gRPC (Google moderno), JSON-RPC.
- **MCP usa JSON-RPC 2.0** por debajo. `{"jsonrpc":"2.0","method":"tools/call","params":{...},"id":N}` sobre el transporte configurado.
- No es exótico: es "llamar a un método en otro proceso, con convención fija de serialización".

### HITL = Human In The Loop

Tole preguntó. Aclarado con ejemplo concreto aplicable al agente futuro (versión B):
1. Agente detecta 5 productos bajos de stock.
2. Agente crea 3 POs en DRAFT (seguro, no envía nada).
3. **Se para y devuelve al humano**: "He preparado estas 3 POs. ¿Confirmo envío?".
4. Humano responde. Agente ejecuta `sendPurchaseOrder` sobre lo confirmado.

Patrón obligatorio en cualquier agente de producción que mueva dinero, envíe emails, borre datos o interactúe con el mundo real de forma irreversible.

### Transport MCP: stdio vs SSE vs HTTP streamable

Tres opciones físicas de cómo el cliente MCP habla con el servidor:

- **stdio**: cliente arranca al servidor como subproceso y le habla por stdin/stdout. Sin red. Usado cuando el cliente controla el ciclo de vida del servidor (ejemplo: Claude Desktop lanza servidores MCP locales).
- **SSE (Server-Sent Events)**: HTTP con streaming del servidor al cliente. Cliente y servidor procesos independientes.
- **HTTP streamable**: sucesor de SSE en el estándar MCP. HTTP bidireccional. Modelo moderno.

**Decisión**: HTTP streamable, con fallback a SSE si `spring-ai-starter-mcp-client` 2.0.0 no lo soporta todavía (verificar empíricamente en la próxima sesión). Descartado stdio porque el servidor vive en Docker con ciclo propio.

### Guardrails obligatorios del bucle (para versión A)

Los tres que habrá que implementar tras el bucle mínimo:

1. **max_iterations**: `while (iteration++ < MAX)`. Corta si el LLM no converge. Repetir la misma tool con parámetros distintos (`getSupplier(1)`, `getSupplier(2)`) es uso legítimo, no problema. El peligro es no converger, no repetir.

2. **Budget de tokens**: acumular `tokens_in + tokens_out` de cada iteración, abortar si supera umbral. **Importante**: el historial crece no linealmente porque cada vuelta arrastra los `tool_result` anteriores. En 8 vueltas puedes quemar 200k tokens si el historial explota. Es el `ulimit` semántico del LLM.

3. **Timeouts**: por tool call individual y por bucle completo.

### Camino A vs Camino B (bucle explícito vs `ChatClient.tools()`)

Discusión explícita al arrancar. Dos formas de implementar ReAct en Spring AI:

- **Camino A — bucle explícito a mano**: tu código controla el `while`. Envías historial + tools al LLM, parseas respuesta, ejecutas tool si aplica, reinyectas resultado, sigues.
- **Camino B — delegar en `ChatClient.tools()`**: registras tools con `@Tool`, llamas a `chatClient.prompt().tools().call()`, Spring AI gestiona el bucle internamente.

Tole predijo correctamente que A da más control, es más pedagógico, y B es el default de producción típico.

**Decisión**: hacer A completo primero (2-3 sesiones estimadas). Al cerrar A, decidir en caliente si vale una sesión extra de refactor a B para tener el contraste, o pasar al siguiente proyecto del roadmap. **No decidir eso ahora**.

### Catálogo real de tools del `erp-mcp-server`: 13 tools, no 3

Corrección importante en sesión. El prompt de continuación heredado decía "3 tools READ". Tole grepeó con `grep -rn "@Tool" src/main/java --include="*.java"` y salieron **13 tools**:

**Product**:
- READ: `getProduct(id)`, `listLowStockProducts()`, `findProductsByCategory(category)` (categorías en mayúsculas: `COFFEE`, `DAIRY`, `TEA`, `SWEETENERS`, `EQUIPMENT`, `PACKAGING`)
- WRITE: `updateStock(productId, newStock)`

**Supplier**:
- READ: `getSupplier(id)`
- WRITE: `createSupplier(name, contactEmail)`

**PurchaseOrder**:
- READ: `getPurchaseOrder(id)`, `listPurchaseOrdersByStatus(status)` (`DRAFT`, `SENT`, `RECEIVED`, `CANCELLED`)
- WRITE: `createPurchaseOrder(...)` → DRAFT, `sendPurchaseOrder(id)`, `cancelPurchaseOrder(id)`, `receivePurchaseOrder(id)` (incrementa stock)

**Invoice**:
- READ: `listInvoicesByStatus(status)` (`PENDING`, `PAID`)

Total: **6 READ + 7 WRITE**. La existencia de WRITE cambia el diseño del agente y motivó el desdoble A→B.

### Versión A (READ-only) primero, B (READ+WRITE con HITL) después

**Versión A**: escenario canónico "reponer stock". Prompt del usuario: *"Necesito reponer stock. Dime qué productos están por debajo del umbral, agrúpalos por categoría, y para cada uno indícame el proveedor de contacto"*. El agente ejecuta `listLowStockProducts` → `getSupplier(id)` por cada supplier único → responde con texto sintetizado. Termina ahí, sin crear ninguna PO. La agrupación por categoría es cognición del LLM (no requiere tool call — el LLM tiene los datos en el historial de la iteración 1).

**Versión B**: agente prepara POs en DRAFT, se pausa con HITL, humano confirma, agente ejecuta `sendPurchaseOrder` sobre lo confirmado.

Razón del orden: con solo READ los errores del agente son **inocuos** (alucinación, no convergencia, tool mal invocada). Permite iterar sobre el diseño del bucle sin miedo a romper estado del ERP.

### Bootstrap del proyecto con Spring Initializr web

Datos usados en el formulario:
- Maven, Java 21, Jar, Spring Boot 4.1.0 (default de Initializr hoy)
- Group: **`dev.toleflaco`** (corregido tras cazar inconsistencia; ver aprendizaje de proceso más abajo)
- Artifact: `erp-purchasing-agent`
- Package: `dev.toleflaco.erppurchasingagent`
- Dependencies: `Spring Web` (renombrado a `spring-boot-starter-webmvc` en Boot 4.x), `Spring Boot Actuator`, `Anthropic Claude` (`spring-ai-starter-model-anthropic`), `Model Context Protocol Client` (`spring-ai-starter-mcp-client`)

Descargado como ZIP, descomprimido en WSL con:
```bash
cd ~/proyectos
unzip /mnt/c/Users/mtole/Downloads/erp-purchasing-agent.zip
```

**Estructura resultante**:
```
~/proyectos/erp-purchasing-agent/
├── .gitattributes
├── .gitignore
├── .mvn/
├── HELP.md
├── mvnw
├── mvnw.cmd
├── pom.xml
└── src/
```

**Detalle Boot 4.x**: `spring-boot-starter-webmvc` en vez del clásico `spring-boot-starter-web`. Renombrado en 4.x para diferenciarlo de reactive. Es la misma dependencia funcional.

**Detalle test starters**: Initializr generó `spring-boot-starter-actuator-test` y `spring-boot-starter-webmvc-test`. NO generó el `spring-boot-starter-test` general. Puede que Boot 4.x lo haya fragmentado. Si al hacer el primer `mvn compile` o tests fallan JUnit 5 / Mockito / AssertJ, verificar si hay que añadir un starter adicional en la próxima sesión.

## Decisiones lockeadas

1. **Camino A (bucle ReAct explícito a mano) primero**. Camino B (`ChatClient.tools()` con abstracción de Spring AI) queda diferido a decisión en caliente al cerrar A, no ahora.

2. **Versión A del agente (READ-only) antes de B (READ+WRITE con HITL)**. Escenario canónico A: "reponer stock, dime productos bajos + proveedor de contacto". Sin creación de POs.

3. **Transport MCP: HTTP streamable con fallback a SSE**. Verificación empírica de qué soporta `spring-ai-starter-mcp-client` 2.0.0 en la próxima sesión. Descartado stdio (servidor en Docker con ciclo propio).

4. **groupId `dev.toleflaco` como convención de ecosistema**, no `com.mtole`. Consistente con `erp-mcp-server` y (verificar) `document-analyzer-ai`. Un ecosistema con dos groupIds mezclados es molestia gratuita al importar tipos entre proyectos hermanos.

5. **Guardrails obligatorios del bucle**: max_iterations, budget de tokens, timeouts (por tool call y por bucle completo). Implementación después del bucle mínimo funcional, no antes.

6. **Cliente MCP y agente como capas conceptuales distintas** aunque vivan en el mismo JAR. El agente usa al cliente MCP; el cliente MCP no sabe qué es ReAct. Facilita cambiar el pattern (agente) o el protocolo (cliente) de forma independiente en el futuro.

7. **Descriptions de `@Tool` y `@ToolParam` se escriben como prompts para el LLM, no como javadoc técnico**. Calidad de la description = factor de primer orden en si el agente funciona.

8. **`git init` diferido hasta que haya algo mínimo funcional**. No hacer `git init` en un proyecto vacío scaffoldado.

## Deuda / Gotchas al cerrar

1. **`<name>` y `<description>` vacíos en el pom.xml** de `erp-purchasing-agent`. Spring Initializr los omitió (no aparecen en el formulario web actual). Rellenar en la próxima sesión antes del primer commit.

2. **Verificar si falta `spring-boot-starter-test` general** en las dependencias de test para JUnit 5 / Mockito / AssertJ. Boot 4.x fragmentó los test starters. Detectable con el primer intento de test unitario.

3. **Verificar empíricamente qué transport MCP soporta `spring-ai-starter-mcp-client` 2.0.0** (HTTP streamable vs SSE). Probable ruta: buscar propiedades disponibles en `application.properties` con autocompletado del IDE, o inspeccionar `mvn dependency:tree` + navegar el jar. Si no soporta streamable, cae a SSE.

4. **No hay `.git/` en `erp-purchasing-agent`**. Habrá que hacer `git init` cuando exista algo mínimo funcional que commitear (probablemente al cerrar la próxima sesión, con `application.properties` + bucle mínimo + endpoint REST + primer test de humo verde).

5. **Lección heredada del arco Redis a aplicar aquí**: `spring.ai.anthropic.chat.max-tokens=1024` truncaba respuestas, subido a 4096 en `document-analyzer-ai`. Aplicar de entrada 4096 aquí también.

6. **Deuda técnica del arco Redis (informativa, no bloquea el ReAct)**: bug `LlmLoggingAdvisor` × `MessageChatMemoryAdvisor`, defensa markdown fences, `MULTI/EXEC` en `saveAll`, TTL a `application.properties`. Sesiones aparte.

## Frases ⭐⭐⭐ interview-ready

**Qué es ReAct**:
> ReAct es un pattern donde un LLM alterna entre razonamiento y acción en un bucle explícito. Cada iteración del bucle es una llamada al modelo; el modelo decide en cada vuelta si invocar una tool (con nombre y parámetros) o devolver respuesta final. Tu código orquesta el bucle: envía historial + catálogo de tools disponibles al modelo, parsea la respuesta, ejecuta la tool si aplica, añade el resultado como nuevo mensaje en el historial, y vuelve a llamar. La condición de salida no es de negocio, es "el modelo dejó de pedir tools".

**Stateless API y catálogo de tools en cada request**:
> La API de Anthropic es stateless: no recuerda nada entre llamadas. En cada llamada al LLM se envía en el body el system prompt, el historial completo de mensajes acumulados, y el catálogo completo de tools disponibles con sus JSON schemas. El LLM ve una foto completa en cada request; toda la persistencia de contexto es responsabilidad del cliente.

**Roles de mensajes como etiquetas convencionales**:
> Los roles `system`, `user`, `assistant` y `tool_result` en la API son etiquetas convencionales que el cliente pone a los mensajes que envía. No son actores. El "user" no habla con el LLM directamente; es la etiqueta que tu código pone al mensaje que originalmente vino del usuario final vía tu endpoint REST. El LLM solo ve mensajes etiquetados.

**Arquitectura MCP: tres capas**:
> En un agente que consume MCP hay tres capas conceptuales: el agente (lógica del bucle ReAct, dominio propio), el cliente MCP (librería que habla protocolo MCP con el servidor, sabe `tools/list` y `tools/call`, no sabe nada de LLMs), y el servidor MCP (expone tools de un dominio como funciones invocables). Aunque el agente y el cliente MCP puedan vivir en el mismo proceso, son capas separadas: cambiar el pattern del agente no toca al cliente MCP, y cambiar la versión del protocolo no toca al agente.

**Descriptions de `@Tool` como prompts**:
> Las descriptions de `@Tool` y `@ToolParam` NO son javadoc para humanos. Son literalmente el prompt que el LLM lee para decidir cuándo usar cada tool y cómo llamarla. Descriptions genéricas o técnicas ("lowStockList") producen agentes que no invocan tools o las llaman en momentos absurdos. Descriptions bien escritas empiezan por un verbo claro, explican cuándo aplica, y describen los parámetros con contexto de negocio. La calidad de estas descriptions es factor de primer orden en si un agente funciona en producción.

**Guardrails obligatorios de un agente**:
> Un agente en producción necesita tres guardrails mínimos: max_iterations para cortar bucles no convergentes, budget de tokens acumulado a lo largo del bucle porque el historial crece no linealmente (cada iteración arrastra los tool_results anteriores), y timeouts tanto por tool call individual como por bucle completo. Sin estos guardrails, un LLM en bucle patológico puede quemar cientos de miles de tokens en minutos.

**HITL en acciones irreversibles**:
> Human In The Loop es el patrón donde un agente automatizado se detiene en un checkpoint crítico y pide confirmación al humano antes de proceder con una acción irreversible o costosa. La regla operativa: automatiza todo lo que es seguro y reversible; introduce HITL antes de gastar dinero real, enviar comunicaciones externas, o modificar estado que no se pueda revertir trivialmente. Se implementa devolviendo control al cliente con el estado del agente serializado, esperando respuesta humana, y reanudando el bucle con la decisión incorporada al historial.

**Transports MCP**:
> El protocolo MCP soporta tres transports principales: stdio para servidores locales lanzados como subproceso por el cliente, SSE como transport HTTP legacy con streaming servidor-a-cliente, y HTTP streamable como sucesor moderno bidireccional. Para servidores dockerizados con ciclo de vida independiente, stdio queda descartado; entre SSE y HTTP streamable, elegir el más moderno soportado por la versión del cliente MCP disponible.

## Estado de la app y roadmap

- **`document-analyzer-ai`**: Proyecto 1 del roadmap AI Engineer sustancialmente completo. Cache-aside en ambos endpoints, prompts robustecidos, max-tokens dimensionado. Bloque 4 Redis cerrado 100%.
- **`erp-mcp-server`**: cerrado en S15 con 13 tools. Servidor MCP listo para ser consumido.
- **`erp-purchasing-agent`**: **arrancado en esta sesión**. Scaffolding descargado, dependencias correctas, sin código propio.
- **Roadmap AI Engineer**: Fase 2.2 en curso. Sesiones estimadas para versión A: 2-3. Versión B posterior: 1-2. Total del arco ReAct: 3-5 sesiones.
- **Roadmap post-Git/Bash (AWS)**: Sesión 12 de Terraform imports en curso, gestionada en chat separado. Alterna por rotación día sí/no con el eje AI.

## Correcciones y aprendizajes de proceso

- **Error importante mío 1: catálogo de tools desactualizado**. Mi memoria y el prompt de continuación heredado decían "3 tools READ". Tole grepeó y salieron 13 tools (6 READ + 7 WRITE). Reconocido explícitamente, retirado sin drama, redimensionado el scope del agente en consecuencia (motivó el desdoble A → B). Aprendizaje: **cuando el usuario diga "hay más X, no solo N", verificar antes de defenderme**. El código real siempre gana a la memoria.

- **Error importante mío 2: groupId sugerido incorrecto**. Propuse `com.mtole` (según memoria). Tole cazó y preguntó "seguro?? no sería dev.toleflaco???". Le di la razón. Aprendizaje: **memoria de convenciones envejece; verificar contra el pom.xml del proyecto hermano más reciente cuando importa la consistencia de ecosistema**.

- **Momento de vulnerabilidad de Tole ("me siento inútil por preguntar HITL")**. Tole comentó que "preguntar todo el tiempo" le hacía sentir inútil y ligó eso al trabajo actual de pescadería. Respondí cortando el hilo con dos frases directas: preguntar no es sinónimo de inutilidad, y trabajar en pescadería mientras te reposicionas no es evidencia de inutilidad — son cosas distintas. Sin dramatizar. Después seguimos con lo técnico sin más. Aprendizaje mutuo: la señal ("me siento inútil") merece registro pero no expansión emocional; una pausa breve, honesta y calibrada, y volver al trabajo, es lo que sirve.

- **Feedback de Tole sobre metacomentarios**: al final de la sesión, Tole pidió explícitamente que no haga comentarios tipo "es normal preguntar" cuando pregunte el significado de una sigla. Aceptado. **Regla lockeada en el prompt de continuación**: cuando Tole pregunta qué significa un término inglés o una sigla, responder brevemente y seguir. Es información, no terapia.

- **Sócrates funcionó bien con petición explícita de empujón**. En un punto Tole se bloqueó reintentando el flujo del bucle ReAct ("no sé como sería el bucle lo siento, voy a inventarmelo"). Petición explícita → empujón dado con iteraciones ejemplificadas paso a paso. Es exactamente el patrón acordado del prompt de arranque. Sistema funcionando.

- **Predicciones acertadas de Tole**: predijo correctamente Camino A > Camino B en las tres dimensiones (control, pedagogía, uso típico en producción). Predijo correctamente que "mi código le dice las tools que hay en mcp_server" y que "habrá una opción de listar las tools". Modelo mental de arquitectura sólido a pesar de ser primer contacto con MCP como cliente.

- **Corte del intento CLI a favor de web**: propuse construir un `curl` a `starter.zip` de Initializr como ejercicio. Tole cortó con "vamos a hacerlo por la web de sprintinitzlr". Buena decisión suya: más rápido para arrancar, la CLI es aprendizaje de segundo orden que se ve otro día si apetece. **Aprendizaje**: la eficiencia de arranque a veces pesa más que el ejercicio pedagógico. Respetar cuando el usuario corta.

- **Densidad conceptual alta sin código**: la sesión asentó ~8-10 conceptos nuevos importantes (ReAct pattern, roles de mensajes, 3 actores, descubrimiento de tools, descriptions como prompts, RPC, HITL, transports MCP, guardrails, sobrecarga de scope A→B). Cero código propio escrito. Para ser primera sesión de un tema nuevo, es el balance correcto: mejor asentar bien el modelo mental que producir 100 líneas que haya que rehacer por diseño equivocado.

## Próxima sesión

**Sesión 18 — `erp-purchasing-agent` parte 2: configuración y bucle mínimo**.

Objetivo: dejar el proyecto arrancable end-to-end con un bucle ReAct mínimo (sin guardrails todavía) que ejecute al menos una iteración con tool call real contra el `erp-mcp-server`.

**Orden propuesto** (a discutir al arrancar):

1. Housekeeping del pom: rellenar `<name>` y `<description>`. Verificar si falta `spring-boot-starter-test` general.
2. `application.properties`: API key Anthropic (via env var), modelo, `max-tokens=4096`, configuración MCP client apuntando al `erp-mcp-server` (verificar transport streamable vs SSE).
3. Levantar `erp-mcp-server` en Docker (`docker compose up -d`) y verificar puerto expuesto.
4. Primer `mvn compile` verde.
5. Diseño socrático de la clase `ReActAgent` (probablemente `@Service` con método `String run(String userPrompt)`).
6. Implementación del bucle mínimo sin guardrails.
7. Endpoint REST simple `POST /agent/run` para probar con `curl`.
8. Primer test de humo manual con el prompt del escenario "reponer stock".

**No en la próxima sesión**: guardrails formales (max_iterations, budget, timeout), tests con `MockChatModel`, observabilidad estructurada, `git init` (solo cuando haya algo mínimo funcional al cerrar la sesión).

**Alternativas si Tole prefiere cambiar de eje ese día**:

- AWS Sesión 12-D (Terraform import de Route Tables y Security Groups). Continuación natural del eje AWS.
- Sesión de re-lectura guiada de tema oxidado (MapStruct propuesto como primera vez). Cuando la energía esté baja.
- Deuda técnica del arco Redis (LlmLoggingAdvisor × MessageChatMemoryAdvisor). 45-60 min.

Ver también `PromptContinuacion-S16-ReActAgent-parte2-2026-08-20.md` generado al cierre de esta sesión.
