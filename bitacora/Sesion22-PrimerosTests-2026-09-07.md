# Sesion 22 — Primeros tests unitarios del `ReActAgent`

**Fecha**: lunes, 7 de septiembre de 2026.
**Duracion aproximada**: ~2h30min (arranque 06:40, cierre en torno a las 09:00-09:15).
**Proyecto**: `erp-purchasing-agent`.
**Estado al cierre**: HEAD en `5cf5099`, `origin/main` sincronizado, dos commits nuevos respecto a S21.

## Objetivo de la sesion

Arrancar los tests unitarios del `ReActAgent`. Hasta ahora toda la verificacion habia sido smoke tests empiricos (`curl` contra API real de Anthropic + MCP server real). Se necesitaba una capa de tests deterministas, sin consumo de tokens, ejecutables en segundos.

Objetivo mas concreto: **primer test verde**. No cubrir todos los escenarios en una sesion — romper el hielo con el escenario mas simple (respuesta directa sin tool calls) y dejar la infraestructura de test lista para replicar patrones en las siguientes sesiones.

## Decisiones cerradas

Cuatro decisiones tecnicas ordenadas antes de teclear codigo:

**Decision 1 — framework de mocking**: Mockito con estilo BDDMockito. Se descarto la opcion de utilidad oficial de Spring AI tras verificar empiricamente que `spring-ai-test:2.0.1` no incluye `TestChatModel`, `MockChatModel` ni equivalente. Se descarto `FakeChatModel` custom por overhead innecesario cuando construir el `ChatResponse` a mano se necesita igualmente independientemente del framework.

**Decision 2 — mockeo del `ToolCallingManager`**: inyeccion como dependencia (7 → 8 params en constructor). Bean autoconfigurado por `ToolCallingAutoConfiguration` de Spring AI, con `@ConditionalOnMissingBean`, mejor construido que el `ToolCallingManager.builder().build()` local que se usaba antes (trae `ObservationRegistry`, `ToolExecutionExceptionProcessor` configurable). Se descarto refactor a `@ConfigurationProperties` porque el umbral fijado era 10+ params.

**Decision 3 — testeo del guardrail de duracion**: inyeccion de `java.time.Clock` (8 → 9 params). Se descarto valor bajo de config (fragil, depende de velocidad de CPU), se descarto no cubrir (incompleto, contradice el espiritu de S22).

**Decision 4 — estructura de tests**: un solo fichero `ReActAgentTest.java` en `src/test/java/dev/toleflaco/erp_purchasing_agent/agent/`, JUnit 5 + `MockitoExtension`, sin `@SpringBootTest`, instanciacion `new ReActAgent(...)` a mano.

## Conceptos cerrados y verificados con Tole en S22

### 1. Descubrimiento sobre `spring-ai-test` de Spring AI 2.0.x

Investigacion empirica en tres pasos que quedo como paradigma de trabajo:

- **Paso 1**: `find ~/.m2/repository/org/springframework/ai -name "*.jar"` listo todos los modulos ya descargados por Maven. Ninguno se llamaba `spring-ai-test`.
- **Paso 2**: `zipgrep "TestChatModel\|MockChatModel"` dentro de los sources jars. Cero matches.
- **Paso 3**: Maven Central mostro cuatro artifacts `spring-ai-test`, pero solo `org.springframework.ai:spring-ai-test:2.0.1` era oficial (los otros: fork de terceros con groupId `io.github.pig-mesh.ai`, groupId squatter `io.springboot.ai`, proyecto personal `io.github.rifatcakir`). Se descargo con `./mvnw dependency:get -Dartifact=...` sin declararlo en `pom.xml` y se inspecciono su contenido con `jar tf`.

**Contenido real de `spring-ai-test:2.0.1`**: clases abstractas para integration tests (`AbstractToolCallingAdvisorIT`, sufijo `IT` de Maven Failsafe), un `MockWeatherService` que es mock de **tool** (no de LLM), un `AudioPlayer` irrelevante, utilidades de escape de llaves (`CurlyBracketEscaper`), `AbstractChatOptionsTests`. **Ningun mock del `ChatModel`**. La libreria oficial de test de Spring AI no cubre este caso de uso.

Consecuencia practica: para tests unitarios del bucle ReAct, Mockito puro es la respuesta. La comunidad Spring AI espera que uno mockee el `ChatModel` a mano construyendo el `ChatResponse` con la estructura correcta.

Version instalada del proyecto es Spring AI 2.0.0. `spring-ai-test` aparecio por primera vez en 2.0.1 (patch retrocompatible). No merecio la pena mezclar versiones o subir el proyecto entero para una libreria que no aporta lo que buscabamos.

### 2. `Clock` inyectado y el trade-off de monotonia

Cambio de mentalidad al migrar de `System.nanoTime()` a `clock.instant()`:

**`System.nanoTime()`**: contador monotono garantizado por especificacion JVM. Solo mide diferencias dentro del mismo proceso. Su origen es arbitrario (no epoch). Su virtud: no le afectan cambios del reloj del sistema (NTP, cambio horario, ajuste manual, VM pausada). Su limitacion: no es testeable — no hay forma de inyectar un valor controlado desde fuera.

**`clock.instant()`**: lee el reloj real del SO. Ligado a la hora de pared. Sujeto a saltos hacia atras por NTP, cambio horario, ajustes manuales, VM pausada. Pero **inyectable, por tanto testeable**.

`Clock` no expone un equivalente a `nanoTime()`. Es una abstraccion sobre "que hora es", no sobre "cuanto ha pasado". Al migrar, el modelo mental cambia:

```java
// ANTES (nanoTime, no testeable)
long start = System.nanoTime();
long elapsedMs = (System.nanoTime() - start) / 1_000_000;

// DESPUES (Clock, testeable)
Instant start = clock.instant();
long elapsedMs = Duration.between(start, clock.instant()).toMillis();
```

**Trade-off aceptado**: se pierde la garantia de que `elapsedMs` sea no-negativo en el caso patologico raro (NTP saltando hacia atras dentro de la ventana de ejecucion). Probabilidad real en un agente con runs de 5-30 segundos con NTP configurado sensatamente: muy baja. La testeabilidad vale mas.

**Regla mental derivada**:
- Medir duraciones puras sin necesidad de testear → `System.nanoTime()`.
- Medir duraciones que necesitas testear → `Clock` inyectado, aceptando el trade-off.
- Guardar timestamps para persistir o mostrar ("cuando ocurrio esto") → `Instant.now()` o `clock.instant()`, nunca `nanoTime()` (que no representa una hora real).

En una entrevista tecnica, ante "¿por que `System.nanoTime()` y no `System.currentTimeMillis()` para medir duraciones?", la respuesta es una palabra: **monotonia**.

**Bean `Clock` en Spring**: Spring no autoconfigura un `Clock`. Al anadir `Clock` como parametro del constructor sin mas, la aplicacion no arranca (`NoSuchBeanDefinitionException`). Se creo `config/TimeConfiguration.java` con un `@Bean Clock clock() { return Clock.systemUTC(); }`. Se prefiere UTC sobre `systemDefaultZone()` por convencion de servidor y para evitar problemas al desplegar en otras zonas horarias.

**Tecnica de test descubierta pero aun no aplicada** (queda para S23): Mockito con `thenReturn` encadenado permite controlar la evolucion del tiempo entre llamadas al clock. Para el escenario de guardrail de duracion, algo como:

```java
given(clock.instant()).willReturn(
    Instant.parse("2026-01-01T00:00:00Z"),  // start
    Instant.parse("2026-01-01T00:00:00Z"),  // primer check, no dispara
    Instant.parse("2026-01-01T00:01:10Z")   // segundo check, +70s, dispara
);
```

En este primer test (happy path sin tools) el clock se dejo como `Clock.fixed(...)` porque el tiempo no importa para el escenario.

### 3. Regla: codigo real > documentacion > tutoriales

Locked en S22 como paradigma de decision para librerias:

- El codigo (`jar tf` para inventario, `-sources.jar` para leer implementacion) es la verdad.
- La documentacion oficial puede estar desactualizada respecto al codigo publicado, o describir features de `main` no liberadas.
- Los tutoriales de terceros (Baeldung, Medium, StackOverflow) suelen estar desactualizados o mezclar versiones.

**Consecuencia practica**: cuando toca decidir "¿existe X en la libreria?" o "¿cual es la firma de Y?", el atajo es `jar tf` + `zipgrep` + `-sources.jar`. No Google. La investigacion empirica dio hallazgos criticos en esta sesion que ningun tutorial habria contado bien:
- `spring-ai-test:2.0.1` no incluye mock del `ChatModel`.
- `Usage` es interface, no clase concreta con constructor publico.
- El bean `ToolCallingManager` existe autoconfigurado con `@ConditionalOnMissingBean` (grep en `ToolCallingAutoConfiguration.java`).

Herramientas clave descubiertas o consolidadas en S22:
- `find ~/.m2/repository/...` para inventario de jars descargados.
- `jar tf <jar>` para listar contenido.
- `jar xf <jar> <path>` para extraer un fichero concreto.
- `zipgrep "pattern" <jar>` para buscar dentro del zip sin descomprimirlo (viene con `unzip`).
- `./mvnw dependency:get -Dartifact=...` para descargar un jar al repo local sin declararlo en el pom.
- `./mvnw dependency:tree | grep -E "..."` para verificar transitivos.

### 4. Disciplina de smoke test empirico despues de refactor

Los tres refactors del paso 2-3 (crear `TimeConfiguration`, inyectar `ToolCallingManager`, inyectar `Clock`) eran "no cambia comportamiento". Un smoke test empirico con `curl` real verifica esa promesa. Detecta problemas antes de escribir el primer test — evita pelearse con un test que falla por dos razones distintas (bug en refactor + bug en test).

Verificacion cruzada mental al ejecutar el `curl`:
- `duration_ms` un numero razonable (no 0) → confirma que `Duration.between()` funciona.
- `iterations` en el rango esperado → confirma que el bucle sigue funcionando.
- Formato del log intacto → confirma que no se rompio nada perimetral.

Resultado en S22: HTTP 200, `iterations=2`, `tokens_total=5028`, `duration_ms=7411`, `cost_usd=0.024624`. Todo verde. Refactor validado.

## Bugs diagnosticados en S22

**Bug 1** — arranque de smoke test sin contenedor MCP. El `curl` de verificacion post-refactor requiere el `erp-mcp-server` corriendo en Docker. En tests unitarios reales no hace falta (todo mockeado), pero en smoke test si. El prompt de continuacion de S21 lo decia explicitamente y aun asi se olvido. Anotado como recordatorio para futuras sesiones.

**Bug 2** — declaracion duplicada de firma de metodo. Al pegar el scaffold del test se quedo la firma `private ChatResponse buildResponseWithoutToolCalls(...)` dos veces, una anidada dentro de otra. Java no permite metodos anidados (a diferencia de Python). Detectado antes de compilar.

**Bug 3** — `builder()` sin `.build()`. `ChatGenerationMetadata.builder().finishReason("end_turn")` devuelve el `Builder`, no el `ChatGenerationMetadata`. Faltaba el `.build()` final.

**Bug 4** — `ZoneId.of("Europe/Madrid")` en test de `Clock.fixed`. Acopla el test al contexto local. Convencion en tests: `ZoneOffset.UTC`. Ademas coincide con el `Clock.systemUTC()` de produccion. Corregido.

**Bug 5** — hipotesis inicial mal calibrada sobre tipo de retorno de getters de `Usage`. Se dudo si eran `Integer` o `Long`. Verificado con `jar xf` + `cat` que son `Integer`. Sin bug real gracias a autoboxing, pero verificacion evito problema potencial de `ClassCastException` en runtime.

**Bug 6** — primer `zipgrep` con doble filtro fallo silenciosamente. `zipgrep "..." <jar> | grep "Usage.java"` no devolvia nada. Segundo intento con solo `jar tf | grep -i "/usage.java"` si funciono. El primer patron encadenado tenia algun conflicto de formato. Anotado.

## Deuda tecnica anadida en S22

- Warning de Mockito sobre self-attaching como Java agent. Futuras versiones del JDK deshabilitaran carga dinamica de agents. Fix: anadir `-javaagent:${settings.localRepository}/net/bytebuddy/byte-buddy-agent/.../byte-buddy-agent.jar` en config del plugin surefire. **No bloquea** ahora.
- `run()` devuelve `String` en vez de un `AgentRunResponse` completo con `iterations`, `tokens_total`, `duration_ms`, `cost_usd`. Estos datos se calculan y loguean pero no se exponen. Para tests que quieran asertar sobre metricas (no solo texto), habra que spy'ear logs (fragil) o refactorizar para devolver un objeto rico. No en S22, quiza en S23 o mas adelante segun necesidad de los tests restantes.
- Constructor de `ReActAgent` con 9 parametros. Umbral de refactor a `@ConfigurationProperties` (10+) sigue sin cruzarse pero cada vez mas cerca. Anadir HITL en S23 puede empujar al umbral.

## Estado del proyecto `erp-purchasing-agent`

**Ubicacion**: `~/proyectos/erp-purchasing-agent/`.

**Estado git**: HEAD `5cf5099`, working tree clean, sincronizado con `origin/main`. Seis commits totales:
- `852b592` — bootstrap (S18).
- `0f192af` — ReAct loop minimo (S19).
- `c614bcd` — per-iteration logging (S20).
- `91dfddf` — guardrails (S21).
- `ffeee4b` — refactor for testability: inyectar `ToolCallingManager` + `Clock` + nueva `TimeConfiguration` (S22).
- `5cf5099` — primer test unitario del ReAct loop happy path sin tool calls (S22).

**Estructura al cerrar**:
```
~/proyectos/erp-purchasing-agent/
├── pom.xml
├── src/
│   ├── main/
│   │   ├── java/dev/toleflaco/erp_purchasing_agent/
│   │   │   ├── ErpPurchasingAgentApplication.java
│   │   │   ├── agent/
│   │   │   │   └── ReActAgent.java              (constructor 9 params)
│   │   │   ├── config/                          (NUEVO en S22)
│   │   │   │   └── TimeConfiguration.java       (Bean Clock.systemUTC())
│   │   │   ├── controller/
│   │   │   │   └── AgentController.java
│   │   │   ├── dto/
│   │   │   │   ├── AgentRunRequest.java
│   │   │   │   └── AgentRunResponse.java
│   │   │   └── exception/
│   │   │       └── GuardrailExceededException.java
│   │   └── resources/
│   │       └── application.yml
│   └── test/
│       └── java/dev/toleflaco/erp_purchasing_agent/
│           ├── ErpPurchasingAgentApplicationTests.java   (bootstrap, sin tocar)
│           └── agent/                                     (NUEVO en S22)
│               └── ReActAgentTest.java                    (1 test verde)
```

**Constructor de `ReActAgent`** (9 parametros, en orden):
```
ChatModel chatModel,
List<McpSyncClient> mcpClients,
@Value(input-per-mtok) double inputCostPerMillionTokens,
@Value(output-per-mtok) double outputCostPerMillionTokens,
@Value(max-iterations) long maxIterations,
@Value(max-tokens-budget) long maxTokensBudget,
@Value(max-duration-ms) long maxDurationMs,
ToolCallingManager toolCallingManager,    // NUEVO S22
Clock clock                                // NUEVO S22
```

**Verificacion tests**: `./mvnw test -Dtest=ReActAgentTest` → verde, ~1.3s, cero llamadas a Anthropic API.

## Sesion emocional / de trabajo

Arranque a las 06:40 en lunes por la manana, energia OK. Silencio de Coditramuntana desde el 1-sep sin novedades. Sin rehacer plan de candidaturas en esta sesion.

Densidad de decisiones socraticas alta (cuatro decisiones tecnicas cerradas + subdecisiones dentro de cada una). Regla de "una decision por mensaje" en terreno nuevo respetada, salvo en el pack final de `Clock` donde se agruparon cuatro conceptos consecutivos (que es Clock, que metodo usar, de donde sale el bean, como se controla en tests) — aceptable porque los conceptos son secuenciales y Tole confirmo comprension entre cada uno.

Intento de saltar directamente a `@ConfigurationProperties` + Claude Code cortado a tiempo (scope creep + delegacion incorrecta del aprendizaje). Buen ejemplo de la contradiccion abierta que la regla socratica autoriza.

Cierre con dos commits y push, S22 sellada en `origin/main`. Momentum bueno para arrancar S23 con los seis escenarios restantes.
