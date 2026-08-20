# Sesión 18 — 2026-08-20 (ReAct agent parte 2, configuración y handshake MCP)

**Continuación directa de la Sesión 17 del mismo día. Fase 2.2 del roadmap AI Engineer — Proyecto 2: `erp-purchasing-agent`. Sesión de infraestructura pura: pom limpio, `application.yml` configurado, transport MCP verificado empíricamente, handshake cliente↔servidor confirmado.**

Duración efectiva: ~1h, jueves tarde tardía, tercer chat del día. Energía media, sostenida. Sesión acotada explícitamente a una hora al arrancar ("vamos a seguir una hora más"). Scope propuesto y aceptado: pasos 1-4 del prompt de continuación (pom, `application.yml`, arrancar `erp-mcp-server`, `mvn compile`), con paso 2 identificado como el que más podía atascarse (config del MCP client, transport a verificar).

Sesión **muy pequeña de código, muy alta en verificación empírica y aprendizaje de patrones de introspección**. Cero commits, cero código Java propio escrito. Sí: pom limpio, `application.yml` funcional, handshake MCP verificado con logs empíricos concretos.

## Repo state al cerrar

- **`~/proyectos/erp-purchasing-agent/`**: **NO es repo git todavía** (sin `.git/`). Working tree ampliado respecto a S17: `pom.xml` limpio (basura de Initializr fuera, `<name>` y `<description>` rellenos), `application.properties` original eliminado, `application.yml` creado y funcional con Anthropic + MCP client + puerto 8082. Sin código Java propio más allá del `ErpPurchasingAgentApplication.java` de Initializr.
- **`~/proyectos/erp-mcp-server/`**: HEAD `ff35c28`, working tree clean. Levantado durante la sesión (`docker compose up -d`), parado al cierre (`docker compose down`).
- **`~/proyectos/document-analyzer-ai/`**: HEAD `8e03afb`, working tree clean. No tocado.

Contenedores Docker todos parados al cerrar. Proceso Spring Boot del agente parado con `Ctrl+C`.

## Commits generados en la sesión

**Ninguno**. El proyecto sigue sin ser repo git. Pendiente para próxima acción (fuera de esta sesión técnica): `git init` + creación de repo en GitHub + primer commit del bootstrap configurado.

## Conceptos cubiertos

### Housekeeping del `pom.xml` generado por Initializr

Spring Initializr moderno genera metadata que solo tiene sentido si el artefacto se publica a Maven Central o similares:

- `<url/>`: URL del proyecto (web pública).
- `<licenses/>`: declaración de licencia (Apache 2.0, MIT, etc.).
- `<developers/>`: mantenedores con nombre, email, rol.
- `<scm/>`: Source Control Management (info del repo git, `connection`, `developerConnection`, `url`).

**Regla lockeada**: en proyectos personales que no se publican como librería, esta metadata es ruido y se borra. `<name>` y `<description>` sí se rellenan porque los usan varias herramientas (IDE, spring-boot-actuator, logs de arranque).

Aplicado: pom queda con `<name>erp-purchasing-agent</name>` y `<description>ReAct agent consuming erp-mcp-server tools via MCP client</description>`, y los cuatro bloques de metadata pública borrados.

### Verificación empírica de dependencias transitivas — `mvn dependency:tree`

Predicción explícita de Tole antes de verificar: los starters de test específicos de Boot 4.x (`spring-boot-starter-actuator-test`, `spring-boot-starter-webmvc-test`) traen JUnit / Mockito / AssertJ por transitividad sin necesidad de añadir `spring-boot-starter-test` general.

Verificación con `mvn dependency:tree`. Predicción confirmada:

```
+- org.springframework.boot:spring-boot-starter-actuator-test:jar:4.1.0:test
|  +- org.springframework.boot:spring-boot-starter-test:jar:4.1.0:test
|  |  +- org.assertj:assertj-core:jar:3.27.7:test
|  |  +- org.junit.jupiter:junit-jupiter:jar:6.0.3:test
|  |  +- org.mockito:mockito-core:jar:5.23.0:test
|  |  +- org.mockito:mockito-junit-jupiter:jar:5.23.0:test
|  |  +- org.hamcrest:hamcrest:jar:3.0:test
|  |  +- org.awaitility:awaitility:jar:4.3.0:test
|  |  +- org.springframework:spring-test:jar:7.0.8:test
|  |  ...
```

**Hallazgo**: `spring-boot-starter-test` sigue existiendo en Boot 4.x y sigue trayendo el kit completo (JUnit 5, Mockito, AssertJ, Hamcrest, Awaitility, jsonassert, spring-test, xmlunit). La "fragmentación" de Boot 4.x no es tan brutal como parecía: `spring-boot-starter-test` viene "por debajo" via los starters de test específicos, no está declarado directamente en el pom que genera Initializr.

**Consecuencia práctica**: si en una refactorización futura se quitan los starters específicos (actuator-test, webmvc-test), se perdería el kit de test transitivo. Habría que añadir explícitamente `spring-boot-starter-test`. Anotación mental para futuras refactorizaciones.

**Regla lockeada**: las dependencias transitivas son cómodas pero opacas. Antes de asumir qué trae un starter, verificar con `mvn dependency:tree` y leer el output.

**Nota adicional**: JUnit Jupiter 6.0.3 (versión mayor recién saltada, salto de 5.x → 6.x) y Mockito 5.23.0 son las versiones que trae Boot 4.1. No sorprenderán si aparecen incompatibilidades con librerías de test antiguas.

### Introspección de propiedades de autoconfiguración de Spring AI — patrón lockeado

Problema recurrente: la documentación online de Spring AI 2.0.0 (recién salida) tiene mucho tutorial obsoleto que usa rutas de propiedades deprecadas o inexistentes. Confiar en tutoriales es fuente de bugs silenciosos.

**Patrón lockeado para descubrir propiedades reales sin adivinar**:

1. Localizar el jar de autoconfiguración correspondiente en `~/.m2/repository/`.
2. `jar tf <jar_binario>.jar | grep -i propert` para listar las clases `*Properties*` del jar. Nota: `find *.class` NO funciona directamente sobre `.m2/` porque Maven guarda los jars empaquetados, no explotados.
3. Si hay un jar de sources aparte (`-sources.jar`), extraerlo en `/tmp` con `jar xf`.
4. `cat` sobre el `.java` real de la clase de properties.
5. Leer el `CONFIG_PREFIX` (una constante `public static final String`) y los campos anotados con `@ConfigurationProperties` para saber la ruta YAML exacta y los defaults reales (o la ausencia de defaults).

Aplicado en esta sesión para dos clases distintas: `AnthropicChatProperties` y `McpStreamableHttpClientProperties`. Tercera verificación indirecta: listar `*Properties*` del jar `spring-ai-autoconfigure-mcp-client-common-2.0.0` para confirmar qué transports soporta la librería (aparecieron `McpStdioClientProperties`, `McpSseClientProperties`, `McpStreamableHttpClientProperties`).

**Regla lockeada**: cuando la documentación es dudosa o el ecosistema es joven, la fuente de verdad es el `.java` de la clase `@ConfigurationProperties`. Adivinar rutas es apostar; introspectar el jar son tres comandos.

### Hallazgos sobre `AnthropicChatProperties` en Spring AI 2.0.0

Tres hallazgos al leer el fuente:

1. **`CONFIG_PREFIX = "spring.ai.anthropic.chat"`**. La ruta correcta para el modelo es `spring.ai.anthropic.chat.model` (**sin nivel `options` intermedio**).

2. **La ruta con `.options.` está deprecada**. La clase interna `Options` está entera marcada `@Deprecated(since = "2.0.0", forRemoval = true)` y cada getter tiene `@DeprecatedConfigurationProperty(replacement = "spring.ai.anthropic.chat.XXX")`. Muchísima documentación y tutoriales que se encuentran online usan `spring.ai.anthropic.chat.options.model` — funciona con warning pero desaparecerá. Ojo al copiar de tutoriales.

3. **No hay defaults para el modelo**. Todos los campos son `@Nullable` sin inicializar:
   ```java
   private @Nullable String model;
   private @Nullable Integer maxTokens;
   ```
   Si no defines `model` explícito, se le pasa `null` al `AnthropicChatOptions.builder()`. El default real (si existe) lo pone alguna capa del SDK `anthropic-java-core`. Es caja negra.

**Regla defensiva lockeada**: siempre configurar el modelo explícito en `application.yml`. Depender de defaults ocultos del SDK compromete reproducibilidad y control de costes.

### Colisión de puertos entre proyectos del mismo ecosistema

Detectada durante configuración: el `erp-mcp-server` mapea `8080:8080` en su `docker-compose.yml`, y el agente arrancaba también en 8080 por defecto. Colisión inevitable al levantar ambos.

Dos opciones planteadas:
- **A**: cambiar el puerto del agente en `application.yml`.
- **B**: cambiar el puerto expuesto del contenedor del `erp-mcp-server`.

Tole eligió A con criterio limpio: "toco el proyecto en desarrollo activo, no el estable; tocar el `erp-mcp-server` obligaría a commit ahí y romper working tree clean".

**Regla lockeada**: cuando dos proyectos activos colisionan en un recurso (puerto, path de disco, tópico de Kafka), se toca el proyecto en desarrollo activo, no el estable. Preserva la limpieza histórica del estable y evita commits contaminantes que dificultan luego `git bisect` o revert. Agente movido a 8082.

### Transport MCP verificado empíricamente — STREAMABLE, no SSE

Punto donde hubo confusión y aprendizaje mutuo.

Primer `curl -i http://localhost:8080/mcp` devolvió:
```
HTTP/1.1 400
Invalid Accept header. Expected TEXT_EVENT_STREAM
```

Interpretación mía precipitada: "es SSE". Tole paró la sesión y aportó dato empírico: el `application.properties` del `erp-mcp-server` tiene `spring.ai.mcp.server.protocol=STREAMABLE`. Configuración explícita del servidor > mi inferencia por content-type.

**Aclaración correcta**: el transport HTTP Streamable (sucesor moderno de SSE en el estándar MCP) también negocia con content-type `text/event-stream` en ciertos flujos. El mensaje del `curl` no distingue SSE clásico de HTTP Streamable — solo indica que el endpoint es de stream. La configuración explícita del servidor es la fuente de verdad.

Segundo `curl -H "Accept: text/event-stream"` devolvió:
```
HTTP/1.1 400
Session ID required in mcp-session-id header
```

Confirmación de que el protocolo tiene handshake y session management. **No se habla MCP a mano con curl**; para eso está la librería cliente.

**Regla lockeada**: el transport MCP NO se negocia entre cliente y servidor, se **configura estáticamente en ambos extremos y tienen que coincidir**. Servidor `STREAMABLE` requiere cliente `spring.ai.mcp.client.streamable-http.*`. Servidor STREAMABLE y cliente SSE simplemente no se hablan. Antes de configurar el cliente, leer siempre la configuración explícita del servidor.

### Configuración del `application.yml` — cambio de `.properties` a `.yml`

Debate `properties` vs `yml` planteado por Tole. Argumentos honestos evaluados: `properties` es más simple y difícil de romper (sin indentación); `yml` supera cuando hay tres o más niveles de anidamiento repetidos (Spring AI, MCP client). Decisión de Tole: `yml`, aprovechando proyecto nuevo.

**Regla suave lockeada**: YAML gana en proyectos con configuración jerárquica repetida. Tabs rompen el parser YAML: siempre espacios, dos por nivel.

`application.yml` final funcional:
```yaml
spring:
  application:
    name: erp-purchasing-agent
  ai:
    anthropic:
      api-key: ${ANTHROPIC_API_KEY}
      chat:
        model: claude-sonnet-4-5
        max-tokens: 4096
    mcp:
      client:
        streamable-http:
          connections:
            erp-server:
              url: http://localhost:8080
              endpoint: /mcp
server:
  port: 8082
```

Puntos que se corrigieron en review antes del primer arranque:
- `Erp_purchasing_agent` → `erp-purchasing-agent` (kebab-case coherente con el artifactId y con los proyectos hermanos).
- `{ANTROPIC_KEY}` → `${ANTHROPIC_API_KEY}` (dos errores: faltaba `$` inicial, y el nombre de la env var estaba mal escrito).
- `max-tokens: 10` → `4096` (lección heredada del arco Redis en `document-analyzer-ai`: 1024 truncaba, 4096 sano).
- Ruta del modelo verificada empíricamente sin nivel `options`.

### Opción A vs B para el bloque `streamable-http` — endpoint separado del url

Con el `.java` de `McpStreamableHttpClientProperties` en mano, el `ConnectionParameters` record expone dos campos nullable:
```java
public record ConnectionParameters(@Nullable String url, @Nullable String endpoint) {}
```

Dos hipótesis de uso:
- **Opción A**: todo en `url` (`url: http://localhost:8080/mcp`), sin `endpoint`.
- **Opción B**: separados (`url: http://localhost:8080`, `endpoint: /mcp`).

Tole eligió B. Yo había sugerido probar A primero por "empezar simple". La verificación empírica dio la razón a Tole:
- **Opción A**: la app arrancó sin errores pero **no apareció el log del handshake MCP**. Cliente MCP no conectó al servidor. "Arranca sin error ≠ funciona".
- **Opción B**: la app arrancó y apareció el log clave:
  ```
  i.m.client.LifecycleInitializer : Server response with Protocol: 2025-11-25,
  Capabilities: ServerCapabilities[..., tools=ToolCapabilities[listChanged=true]],
  Info: Implementation[name=erp-mcp-server, ..., version=0.0.1]
  ```

Interpretación del log del handshake:
- `LifecycleInitializer` = componente del cliente MCP que gestiona el arranque de la conexión.
- `Server response with Protocol: 2025-11-25` = handshake exitoso, cliente y servidor negociaron versión del protocolo MCP.
- `Capabilities: ...tools=ToolCapabilities[listChanged=true]` = el servidor declara que expone tools. También declara prompts y resources, pero no los usaremos aquí.
- `Info: Implementation[name=erp-mcp-server, version=0.0.1]` = tu propio servidor identificándose. Ese `name` y `version` vienen del `application.properties` del `erp-mcp-server`.

**Regla lockeada**: "arranca sin error" ≠ "funciona". La verificación real de conectividad es un **log específico de handshake exitoso**: `Connection established` del driver JDBC, `Assigned partitions` del consumer Kafka, `LifecycleInitializer: Server response with Protocol` del cliente MCP. La ausencia de excepción es condición necesaria, no suficiente.

### Warnings inocuos al arrancar el cliente MCP

Aparecieron dos WARN al arrancar el agente que **no son fallo**, solo señales de features MCP opcionales no implementadas:

```
WARN o.s.a.m.a.p.s.SyncMcpSamplingProvider    : No sampling methods found
WARN o.s.a.m.a.p.e.SyncMcpElicitationProvider : No elicitation methods found
```

- **Sampling**: feature MCP donde el servidor puede pedir al cliente que llame al LLM (raro, para agentes chained). No la usamos.
- **Elicitation**: feature donde el servidor puede pedir input al usuario a través del cliente. Tampoco.

Ambos ignorables.

## Decisiones lockeadas

1. **Basura de Initializr fuera en proyectos personales**. Los bloques `<url>`, `<licenses>`, `<developers>`, `<scm>` solo tienen sentido si el artefacto se publica a Maven Central o similar. En proyectos personales se borran. `<name>` y `<description>` se rellenan siempre.

2. **Verificar dependencias transitivas con `mvn dependency:tree` antes de asumir qué trae un starter**. En Boot 4.x, `spring-boot-starter-test` viene por transitividad via los starters de test específicos, no está declarado directamente.

3. **Patrón introspección para descubrir propiedades de Spring AI**: `jar tf` sobre el binario para localizar la clase `*Properties*`, `jar xf` sobre el `-sources.jar` para leer el `.java` real, `cat` para descubrir el `CONFIG_PREFIX` y los defaults. Tres comandos, cero adivinar. Aplica a cualquier librería del ecosistema Spring o compatible.

4. **Modelo Anthropic siempre explícito en `application.yml`**. `AnthropicChatProperties` no tiene defaults visibles; el default real lo pone el SDK `anthropic-java-core` (caja negra). Reproducibilidad y control de costes requieren configuración explícita.

5. **`max-tokens: 4096` por defecto en proyectos con salida estructurada JSON**. Lección heredada del arco Redis: 1024 trunca a mitad de campo con `UnexpectedEndOfInputException`, 4096 cubre casos reales.

6. **Rutas de propiedades sin `.options.` en Spring AI 2.0.0**. La ruta clásica `spring.ai.anthropic.chat.options.model` está deprecada con `forRemoval = true`. Muchos tutoriales están obsoletos.

7. **YAML sobre properties cuando la config tiene tres o más niveles jerárquicos repetidos**. `spring.ai.anthropic.chat.*` se lee de un vistazo en YAML; en properties es repetir el prefijo cada línea.

8. **Colisión de puertos entre proyectos: se toca el proyecto en desarrollo activo, no el estable**. Preserva la limpieza histórica del estable y evita commits contaminantes. Agente en 8082, `erp-mcp-server` intocado en 8080.

9. **Transport MCP se configura estáticamente en ambos extremos y tienen que coincidir**. No hay negociación automática entre cliente y servidor. Leer la config explícita del servidor antes de configurar el cliente.

10. **"Arranca sin error" ≠ "funciona"**. La verificación real de una conexión (BD, cola, servicio, MCP) es un log específico de handshake exitoso. La ausencia de excepción es condición necesaria, no suficiente.

11. **Cliente MCP `streamable-http`: `url` y `endpoint` separados**, no todo junto en `url`. Confirmado empíricamente (opción A arrancó sin error pero sin handshake; opción B arrancó con handshake visible en logs).

## Deuda / Gotchas al cerrar

1. **`erp-purchasing-agent` sigue sin ser repo git**. `git init` + repo en GitHub + primer commit del bootstrap configurado quedan como acción pendiente inmediata al cerrar esta sesión (fuera del scope técnico ya cerrado).

2. **`/actuator/health` del `erp-mcp-server` devuelve 404**. El `erp-mcp-server` no incluye actuator o no lo expone. Deuda técnica menor en ese proyecto: añadir `spring-boot-starter-actuator` y exponer al menos `health` para poder healthcheckear el contenedor de forma limpia. Prioridad baja, sesión aparte.

3. **`document-analyzer-ai` y su `.properties` no tienen el modelo Anthropic explícito**. Usa el default del SDK, que ahora sabemos que es caja negra. Deuda: añadir `spring.ai.anthropic.chat.model=claude-sonnet-4-5` explícito por consistencia y reproducibilidad. Sesión aparte, muy corta.

4. **Deuda técnica arrastrada del arco Redis en `document-analyzer-ai` (no bloquea este proyecto)**:
   - Bug `LlmLoggingAdvisor` × `MessageChatMemoryAdvisor` en `/chat`.
   - Post-procesado defensivo de fences markdown en controllers.
   - `MULTI/EXEC` para atomicidad de `saveAll` en `RedisChatMemoryRepository`.
   - TTL de chat memory + `CvAnalysisCache` a `application.properties`.

## Frases ⭐⭐⭐ interview-ready

**Introspección de dependencias con `.java` de `@ConfigurationProperties`**:
> Cuando la documentación de una librería es dudosa o el ecosistema es joven (Spring AI 2.0.0 recién salido con versionado inestable), la fuente de verdad son las clases `@ConfigurationProperties` del jar de autoconfiguración. `jar tf` sobre el binario para localizar la clase, `jar xf` sobre el `-sources.jar` para leer el `.java` real, `cat` para descubrir el `CONFIG_PREFIX` y los defaults. Tres comandos, cero adivinar. Aplica a cualquier librería Spring o compatible.

**Dependencias transitivas en Boot 4.x**:
> Spring Boot 4.x fragmenta los starters de test en específicos (`spring-boot-starter-actuator-test`, `spring-boot-starter-webmvc-test`), pero `spring-boot-starter-test` sigue existiendo y viene por transitividad arrastrando JUnit 5, Mockito, AssertJ, Hamcrest, Awaitility y spring-test. La consecuencia práctica: si en una refactorización se quitan los starters específicos, hay que añadir explícitamente `spring-boot-starter-test` o pierdes el kit de test. Las dependencias transitivas son cómodas pero opacas — verificar con `mvn dependency:tree`.

**Transport MCP no se negocia**:
> El transport MCP no se negocia entre cliente y servidor: se configura estáticamente en ambos extremos y tienen que coincidir. Servidor con `spring.ai.mcp.server.protocol=STREAMABLE` requiere cliente `spring.ai.mcp.client.streamable-http.connections.*`. Un servidor STREAMABLE y un cliente SSE simplemente no se hablan. Antes de configurar el cliente, leer siempre la configuración explícita del servidor, no inferir por el content-type de la respuesta a un `curl` sin cabeceras.

**"Arranca sin error" ≠ "funciona"**:
> En cualquier configuración de conectividad — base de datos, colas de mensajes, servicios remotos, MCP — el arranque limpio del proceso NO es evidencia de conexión establecida. La verificación real es un log específico de handshake exitoso: la línea del driver JDBC confirmando `Connection established`, el consumer Kafka reportando `Assigned partitions`, el `LifecycleInitializer` de MCP reportando `Server response with Protocol: ...`. La ausencia de excepción es condición necesaria, no suficiente.

**Defaults nullable en Spring AI**:
> En Spring AI 2.0.0 los campos de `AnthropicChatProperties` están todos declarados `@Nullable` sin valores por defecto. Si no configuras `spring.ai.anthropic.chat.model` explícitamente, se pasa `null` al `AnthropicChatOptions.builder()`, y el default real (si existe) lo pone alguna capa del SDK `anthropic-java-core`. Es caja negra. Regla defensiva: siempre configurar el modelo explícito. Compromete reproducibilidad y control de costes depender de defaults ocultos del SDK.

**Colisión de recursos entre proyectos hermanos**:
> Cuando dos proyectos activos colisionan en un recurso (puerto, path de disco, tópico de Kafka), se toca el proyecto en desarrollo activo, no el estable. Preserva la limpieza histórica del estable, evita commits contaminantes que dificultan luego `git bisect` o revert.

**YAML sobre properties para configuración jerárquica repetida**:
> YAML supera a properties cuando la configuración tiene tres o más niveles de anidamiento repetidos (Spring AI, MCP client, autoconfiguración de starters modernos). Un bloque YAML con seis propiedades bajo `spring.ai.anthropic.chat.*` se lee de un vistazo; el equivalente en properties son seis líneas con el prefijo repetido seis veces. Tabs en YAML rompen el parser: siempre espacios, dos por nivel.

**Rutas deprecadas en ecosistemas jóvenes**:
> En librerías jóvenes con salto reciente de versión mayor (Spring AI 1.x → 2.0.0), gran parte de la documentación y tutoriales online usan rutas de propiedades deprecadas o inexistentes. Verificar contra el `.java` de la clase `@ConfigurationProperties` cualquier ruta que se copie de un tutorial. La anotación `@DeprecatedConfigurationProperty(replacement = "...")` en el getter indica la ruta nueva.

## Estado de la app y roadmap

- **`erp-purchasing-agent`**: bootstrap **configurado** + pom limpio + `application.yml` funcional + handshake MCP verificado empíricamente. Cero código Java propio todavía. Cero commits. Aún no es repo git.
- **`erp-mcp-server`**: sin cambios (HEAD `ff35c28`, working tree clean). Confirmado funcionando: 13 tools (6 READ + 7 WRITE), transport STREAMABLE, endpoint `/mcp`, protocolo MCP `2025-11-25`, capabilities completos (tools, prompts, resources).
- **`document-analyzer-ai`**: sin cambios (HEAD `8e03afb`, working tree clean).
- **Roadmap AI Engineer, Fase 2.2 — Proyecto 2 (`erp-purchasing-agent`)**: infraestructura de conectividad resuelta. Toca el core del bucle ReAct en la siguiente sesión.
- **Próxima sesión natural (S19)**: diseño socrático de `ReActAgent` + esqueleto del bucle mínimo (sin guardrails) + endpoint REST `/agent/run` + `LlmLoggingAdvisor` adaptado desde `document-analyzer-ai` + primer test de humo manual con el prompt del escenario "reposición de stock" (READ-only, versión A).

## Correcciones y aprendizajes de proceso

- **Precipitación mía interpretando el error `Expected TEXT_EVENT_STREAM`**: salté a "es SSE" sin verificar la configuración explícita del servidor. Tole paró, aportó el `spring.ai.mcp.server.protocol=STREAMABLE` del `erp-mcp-server`, y corregí. Aprendizaje interno: en respuestas de `curl` a endpoints MCP, verificar la configuración explícita del servidor antes de inferir el transport por el content-type. Buena calibración por parte de Tole: cuestionó cuando la interpretación no le cuadraba en vez de tragar.

- **Predicción explícita antes de verificar (dependencias transitivas)**: Tole predijo correctamente que los starters específicos traían el kit de test. Verificación empírica confirmó. Refuerza el hábito de pedir predicciones antes de ejecutar comandos — construye modelo mental sólido y detecta rápido cuando la intuición falla.

- **Errores de escritura de configuración cazados en review pre-arranque**: `Erp_purchasing_agent` (case inconsistente), `{ANTROPIC_KEY}` (env var mal escrita en dos formas: sin `$` y con nombre erróneo `ANTROPIC_KEY` en vez de `ANTHROPIC_API_KEY`), `max-tokens: 10` (valor sin sentido). Los tres cazados antes de arrancar por review conjunta. Refuerza el valor de pegar la configuración completa para review antes del primer arranque, en vez de escribir y arrancar directamente.

- **Confusión legítima "no sé lo que añadir" en el bloque MCP**: Tole reconoció bloqueo tras leer el fuente de la clase de properties (el `.java` es sencillo pero la aplicación al YAML no era trivial sin haberlo visto antes). Bloqueo genuino, no distracción. Respondido con andamios (dos opciones concretas A/B con sintaxis exacta) y decisión final delegada a Tole. Es exactamente el patrón "sócrates puro con excepción justificada cuando el interlocutor se atasca y pide empujón".

- **Tole eligió Opción B directamente** en la config MCP a pesar de mi sugerencia de "probemos A primero por ser más simple". Aceptado sin re-discutir. Resultó que **B es la correcta**, con verificación empírica clara (log de handshake). Aprendizaje mutuo: mi sugerencia de "empezar por lo simple" fue arbitraria; la elección de Tole por B (separación explícita url/endpoint) coincidió con la semántica real de la librería. Cuando el usuario tiene intuición razonada distinta a la mía, aceptar y verificar empíricamente en vez de imponer mi orden sugerido.

- **Cadena de introspección `jar tf` → `jar xf` → `cat` aplicada dos veces en la misma sesión**: primera para `AnthropicChatProperties`, segunda para `McpStreamableHttpClientProperties`. Interiorizada la herramienta como patrón lockeado. Uso futuro en cualquier momento que surja duda sobre propiedades de librerías del ecosistema Spring o compatible. Aprendizaje: patrones lockeados en una sesión se aplican sin fricción en la misma sesión.

- **Regla `git status` doble** (antes y después de `git add`) mencionada al arrancar pero no aplicada porque no hubo `git add` en esta sesión (sigue sin ser repo git). Se aplicará en la próxima sesión cuando toque hacer `git init` + primer commit.

- **Ansiedad sobre candidaturas mencionada en prompt de continuación heredado y respetada durante la sesión**. Cero fricción emocional en la sesión técnica. Buena calibración.

- **Fatiga natural al final**: Tole marcó explícitamente "hasta aquí es donde ibamos a ver no??" tras el handshake exitoso. Reconocimiento de cierre natural. Decisión buena: cerrar con un hito claro (handshake verificado) en vez de arrancar diseño del `ReActAgent` con fatiga y dejarlo a medias.

- **Confusión sobre generación de notas (mía o suya)**: Tole preguntó "Entiendo que las notas las has generado ya no???". Aclaración: yo no genero nada sin pedir explícito; el archivo que él subió (`SesionAI-2026-08-20-S16-part1.md`) era de una sesión anterior, no de esta. Aclarado y procedido con la generación real de estas notas siguiendo el formato de referencia (`Sesion17-ReActAgent-2026-08-20.md`) que también subió.

## Próxima sesión

**Sesión 19 — `erp-purchasing-agent` parte 3: bucle ReAct mínimo funcional**.

Objetivo: bucle ReAct end-to-end funcionando contra el `erp-mcp-server` con al menos una tool call real, sin guardrails formales todavía. Primer commit del proyecto.

**Orden propuesto** (a discutir al arrancar):

1. **`git init` + `.gitignore` + repo en GitHub + primer commit del bootstrap configurado**. Mensaje bilingüe con separador `---`, cuerpo español sin tildes/ñ. Push inicial.

2. **Diseño socrático de `ReActAgent`**: interfaz mínima (probablemente `String run(String userPrompt)`), lugar en el package (`agent/` o `service/`), inyección de dependencias (`ChatClient` de Spring AI + cliente MCP).

3. **Esqueleto del bucle sin guardrails**: `while` con salida solo por respuesta final del LLM. Sin `max_iterations`, sin budget, sin timeout. Objetivo: ver el flujo end-to-end.

4. **Endpoint REST `POST /agent/run`**: recibe `{"prompt": "..."}`, delega a `ReActAgent.run(...)`, devuelve respuesta. Para probar con `curl`.

5. **`LlmLoggingAdvisor` adaptado** desde `document-analyzer-ai` con logging por iteración (contador + tokens acumulados + tool invocada + args).

6. **Primer test de humo manual**: prompt del escenario "reposición de stock" (READ-only). Verificar qué tools llama el LLM, si converge, si la respuesta final es útil.

**No haremos en la próxima sesión** (van más adelante en el arco A):
- Guardrails formales (`max_iterations`, budget de tokens, timeout).
- Tests unitarios con `MockChatModel`.
- Observabilidad estructurada por iteración.

**Alternativas si Tole prefiere cambiar de eje ese día**:
- AWS Sesión 12-D (Route Tables + Security Groups en Terraform import, chat separado).
- Sesión de re-lectura guiada (MapStruct como tema propuesto, chat separado).
- Deuda técnica del arco Redis: bug `LlmLoggingAdvisor` × `MessageChatMemoryAdvisor` en `/chat` (45-60 min).

Ver también `PromptContinuacion-S19-ReActAgent-parte3-2026-08-20.md` cuando se genere.
