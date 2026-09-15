# Prompt de continuacion — S26-B, `erp-purchasing-agent` diseno detallado del estado persistido de HITL (`AgentRunSession` + `RunStateRepository` + decision sobre `HitlProperties`) y cierre de deuda 17

**Uso**: pegar al inicio de un chat nuevo cuando toque retomar el eje AI. Este chat sigue el roadmap AI Engineer (no el de AWS, que va por otro chat separado). En la sesion S26-A (14-sep manana, ~1h efectiva antes de cerrar por gasto de chat y por fatiga acumulada desde las 5:45) se saldo la deuda 18 con un unico commit empujado a `origin/main` (`4c8057d`): anadir servicio Redis (`redis:7.4-alpine`, sin volumen, sin healthcheck, puerto host 6381) al `docker-compose.yml` del `erp-purchasing-agent`. Los 9 tests no se re-verificaron en S26-A (no habia cambios de codigo Java). Bloque 2 de S26-A (diseno en pizarra) **pospuesto integramente a S26-B**. Deuda 17 **reformulada** y aplazada a S26-B por razonamiento: no habia dependencia `spring-boot-starter-data-redis` en el `pom.xml`, por lo que apagar el autoconfigurador Redis en `application-test.yml` habria sido trabajo especulativo. En S26-B se cerrara junto con la adicion de la dependencia, en el mismo commit.

**S26-B es la sesion de diseno detallado del estado persistido de HITL** (diseno HITL global cerrado en S24). Sesion larga por naturaleza — es diseno en pizarra, no codigo. Cinco piezas a cerrar antes de que se toque una linea de Java en S26-C: (a) shape del `AgentRunSession`, (b) interfaz `RunStateRepository` (o clase concreta directa, por decidir), (c) paquete `hitl` como vertical vs disperso horizontal, (d) decision sobre `HitlProperties` (S26-B o S26-C), (e) cierre de deuda 17 cuando se anada la dependencia.

**S26-B: diseno en pizarra + cierre de deuda 17 en un commit acompanante**. Duracion estimada 1h30 - 2h30. Cinco objetivos:
1. Diseno detallado en pizarra del `AgentRunSession` (shape del POJO persistido en Redis, campos, tipos, serializacion, record vs clase).
2. Interfaz `RunStateRepository` o clase concreta directa — decidir.
3. Paquete `hitl` como vertical de feature vs dispersion horizontal — decidir.
4. Decision sobre `HitlProperties`: S26-B o posponer a S26-C. Discutir al principio.
5. Cuando (y solo si) se anada `spring-boot-starter-data-redis` al `pom.xml` en S26-B, cerrar deuda 17 apagando el autoconfigurador Redis en `application-test.yml` en el mismo commit.

No se toca `ReActAgent` ni `AgentRunResult` ni el controller en S26-B. Esos cambios llegan a partir de S26-C.

---

## Antes de arrancar el chat tecnico

**Sin tarea previa**. Notas de S26-A ya estan en el repo de notas (o subirlas antes de arrancar).

**Verificacion de arranque de entorno**:
- `erp-mcp-server` — **no hace falta arrancarlo** para S26-B (no se corre `contextLoads` con Docker hasta que se cierre eventualmente la deuda 17; incluso entonces la app no se conecta a MCP en el perfil `test`).
- `ANTHROPIC_API_KEY` — **no necesaria** para S26-B (nada de llamadas al LLM).
- Contenedor `erp-purchasing-redis` — no imprescindible para el diseno en pizarra. Si se llega a cerrar la deuda 17 en el mismo chat, hay que asegurarse de que los tests sigan verdes **sin** Redis levantado (ese es el proposito de la deuda). Contenedor se puede levantar y bajar segun convenga.
- Referencia de patron Redis en tests: `~/proyectos/document-analyzer-ai/src/test/resources/application-test.yml` (o el fichero equivalente) puede tener el override — verificar antes de tocar.

---

## Quien soy y como trabajamos

Ver prompt de S25 anterior — reglas identicas. Cambios/notas relevantes desde S25 y S26-A:

- **Regla lockeada en S25 (mantener tipos primitivos en refactor mecanica)**: sigue vigente. Aplicable a S26-B si al cerrar la deuda 17 hay tentacion de "mejorar" algo por el camino en el `application-test.yml` — no toca.
- **Regla lockeada en S25 (sufijo `XxxProperties` en `@ConfigurationProperties`)**: aplicable si se cierra el shape de `HitlProperties` en S26-B.
- **Regla lockeada en S25 (guardar el record completo, no desempaquetar en primitivos)**: aplicable al consumidor de `HitlProperties` cuando llegue.
- **Regla lockeada en S25 (refactor incremental con verificacion entre ficheros)**: aplicable siempre.
- **Regla lockeada en S25 (`records` en Java: coma, sin `final`, sin defaults)**: aplicable al shape del `AgentRunSession` si se decide record.
- **Regla reforzada en S26-A (no cerrar deudas antes de que la dependencia funcional exista)**: cerrar la deuda 17 antes de anadir `spring-boot-starter-data-redis` habria sido trabajo especulativo — el override podria no ser el correcto cuando llegue el momento, o podria necesitar acompanamiento con otras exclusiones. La regla se generaliza: no cerrar una deuda cuando la infra que la justifica todavia no esta introducida. La regla convive con la de "commit at logical arcs" — no son opuestas: hay que decidir el arco correctamente.
- **Regla reforzada en S26-A (commit at logical arcs incluye arcos pequenos)**: la deuda 18 se cerro con un unico fichero nuevo pequeno (`docker-compose.yml` de 6 lineas). No commitear porque "es poco" ensucia el terreno para trabajo futuro. La disciplina se entrena en los casos faciles.

## Estado al cerrar sesion S26-A (14 de septiembre 2026, cierre parcial por gasto de chat y fatiga)

### Deuda 18 saldada

`docker-compose.yml` nuevo en la raiz del proyecto:

```yaml
services:
  redis:
    image: redis:7.4-alpine
    container_name: erp-purchasing-redis
    ports:
      - "6381:6379"
```

Cuatro decisiones cerradas:
1. **Solo servicio `redis`** en el compose. La app se sigue corriendo con `./mvnw spring-boot:run` o desde IntelliJ.
2. **Puerto host 6381** mapeado al 6379 del contenedor, para coexistir sin fricciones con el Redis de `document-analyzer-ai` en el 6379.
3. **Sin volumen**. Argumentos discutidos: YAGNI en dev; `AgentRunSession` ya es efimero por diseno (TTL 24h); el "realismo de produccion" es enganoso porque en prod usarias Redis gestionado (ElastiCache, Upstash), no un contenedor con volumen local; en dev "empezar limpio con `docker compose down && up`" es una operacion que quieres barata.
4. **Sin healthcheck**. Cosmetico para el caso actual con Redis en un contenedor propio sin carga y sin `app` en el compose. Si en el futuro se anade `app` al compose con `depends_on: condition: service_healthy`, se anade entonces.

Verificacion al cierre:
- `docker compose up -d` — contenedor levanto y aparecio en `docker compose ps` como `Up`.
- `docker exec erp-purchasing-redis redis-cli ping` — respondio `PONG`.
- `redis-cli` no esta instalado en el host de Tole; se uso `docker exec ... redis-cli` en vez de instalarlo. Alternativa (instalar `redis-tools`) probada brevemente pero abandonada porque `apt` se colgo. No es bloqueante — el patron `docker exec` es el que se usa habitualmente en Docker.

### Deuda 17 reformulada y pospuesta a S26-B

Enunciado original en el prompt de S25: "`application-test.yml` tendra que apagar Redis. Mismo patron que la deuda 16 saldada."

Razonamiento para posponer: `erp-purchasing-agent` no tiene `spring-boot-starter-data-redis` en el `pom.xml`. Sin esa dependencia, Spring Boot no autoconfigura Redis, y `contextLoads` con perfil `test` no intenta conectarse a nada. Anadir el override en `application-test.yml` antes de que exista la dependencia es trabajo especulativo — el override podria no ser el correcto cuando llegue el momento (Spring Boot podria rechazar la config si no reconoce el namespace, o simplemente ignorarla como propiedad huerfana). Ademas, ensuciaba el arco de commits: hoy cerrar la 17 sin razon funcional, manana revisitar el mismo fichero al anadir la dependencia.

Enunciado reformulado (S26-B): "cuando se anada `spring-boot-starter-data-redis` al `pom.xml` en S26-B, apagar el autoconfigurador Redis en `application-test.yml` en el mismo commit. Verificar que `./mvnw test` sigue 9/9 verde sin Docker levantado."

### Estado del proyecto `erp-purchasing-agent`

**Ubicacion**: `~/proyectos/erp-purchasing-agent/`.

**Estado git**: `main` sincronizado con `origin/main`. Working tree clean tras el commit. Un commit del arco S26-A empujado:

- `4c8057d`: `chore(infra): add Redis service to docker-compose`.

**Verificacion tests al cierre**:
- No se re-verificaron en S26-A. No habia cambios de codigo Java, solo `docker-compose.yml` nuevo. Ultimo estado conocido (cierre de S25): 9/9 verde tanto sin Docker como con Docker.

**`application.yml`** al cierre de S26-A: sin cambios respecto a S25.

**`AgentRunResult`** al cierre de S26-A: sin cambios desde S23-A.
```java
public record AgentRunResult(
    String text,
    long iterations,
    long tokensTotal,
    long durationMs,
    double costUsd
) {}
```

**`AgentController`** al cierre de S26-A: sin cambios desde S23-A. Endpoint unico `POST /agent/run` con codigos 200 / 422.

**`ReActAgent`** al cierre de S26-A: constructor de 6 parametros (firma cerrada en S25).

**`docker-compose.yml`** nuevo (ver bloque arriba).

### Diseno HITL cerrado en S24 (recordatorio)

Guardado en `S24-diseno-HITL.md`. Cinco decisiones cerradas, cero codigo escrito. Resumen ejecutivo — lo que aplica a S26-B esta subrayado:

1. **Que tools requieren HITL**: (a) escrituras con impacto de negocio + (b) lecturas de datos sensibles. RGPD fuera de alcance del agente. — aplicable a `HitlProperties` (lista de tools sensibles).
2. **Como se interrumpe el bucle**: retorno con `status=PAUSED_FOR_APPROVAL` en `AgentRunResult`. — no aplica a S26-B.
3. **Estado entre pausa y reanudacion**: **Redis con TTL 24h refrescable en cada aprobacion. `410 Gone` si expira.** — **base del diseno de `AgentRunSession` en S26-B**.
4. **Endpoints v1**: `POST /agent/run` (200 / 202 / 422) + `POST /agent/run/{runId}/approve` sin body (200 / 202 / 410). — no aplica a S26-B.
5. **Impacto en `ReActAgent`**: refactor a `@ConfigurationProperties` primero (S25, saldado), HITL despues. `HitlProperties` en S26 como dominio propio. — **S26-B: queda pendiente decidir si `HitlProperties` se disena en S26-B junto al `AgentRunSession` o se pospone a S26-C. Anotado como decision temprana a resolver**.

### Estado de repos relacionados (no se tocan en S26-B codigo del `erp-mcp-server`)

- **`~/proyectos/document-analyzer-ai/`**: HEAD `8e03afb`, sin cambios. Sigue siendo referencia para el patron Redis (bloque compose ya usado, pendiente ver el override de `application-test.yml` si lo tiene).
- **`~/proyectos/erp-mcp-server/`**: HEAD `ff35c28`, sin cambios.
- **`~/proyectos/coditramuntana/discography/`**: HEAD `f06b0b4`, sin cambios.

## Que toca en la proxima sesion (S26-B)

**Sesion de diseno en pizarra + posible cierre de deuda 17 al final**. Un unico commit al final (si se llega a anadir la dependencia y cerrar la deuda 17). El diseno del `AgentRunSession` y del `RunStateRepository` **no se codifica** en S26-B — se cierra en pizarra y queda en `S26-B-diseno-estado-persistido.md`. La primera linea de codigo Java nueva del feature HITL llega en S26-C.

### Objetivos

1. **Decision temprana**: `HitlProperties` en S26-B o en S26-C. 5 min de discusion, decidir.

2. **Diseno detallado del estado persistido en pizarra**. Tres bloques a cerrar:
   - **Shape del `AgentRunSession`**: que campos guarda (`runId`, `originalPrompt`, `conversationHistory`, `pendingToolCalls`, `totalPromptTokens`, `totalCompletionTokens`, `iteration`, `createdAt`, ¿algo mas?). Que tipos. Como se serializa a JSON (Jackson estandar del proyecto — snake_case global). Si es record o clase (probablemente clase por Jackson, pero verificar con codigo real).
   - **Interfaz `RunStateRepository` vs clase concreta directa**: metodos minimos (`save(AgentRunSession)`, `findById(String runId)`, ¿algo mas para S26?). Contrato de expiracion — el TTL vive en la implementacion, no en la interfaz. Discutir YAGNI: interfaz solo si hay caso real para dos implementaciones.
   - **Ubicacion en paquetes**: probablemente `dev.toleflaco.erp_purchasing_agent.hitl` como paquete nuevo (feature vertical), o dividir en `hitl.domain` / `hitl.repository`, o dispersar cada pieza en su paquete horizontal existente (`dto`, `config`, `agent`, `exception`, `controller`). Discutir con Tole.

3. **Si se decide (a) en el punto 1**: cerrar shape de `HitlProperties` (lista de tools sensibles, TTL en horas, ¿algo mas?), sin codigo.

4. **Cierre de deuda 17 SOLO si se anade `spring-boot-starter-data-redis` al `pom.xml` en S26-B**. Esto tiene dos condiciones:
   - Que el diseno cerrado justifique anadir la dependencia ya (por ejemplo, para verificar en un test unitario minimo que Jackson serializa el shape decidido).
   - Que quede tiempo y cabeza.
   Si no se anade la dependencia en S26-B, la deuda 17 sigue en su enunciado reformulado para S26-C.

### Elementos a discutir antes de teclear (bloque diseno)

1. **`AgentRunSession` — record o clase?**. Jackson deserializa records desde Spring Boot 2.6+ sin drama, pero conviene verificar con codigo real (`jar tf` sobre `jackson-databind` o probar con test unitario minimo si hay dudas). Preferencia: record por consistencia con `AgentRunResult` y `LlmPricingProperties` / `AgentGuardrailsProperties`. Si Jackson lo soporta limpio, es la eleccion.

2. **Campos del `AgentRunSession`**. Punto por punto. Que necesita el `run()` reanudado para retomar exactamente donde pauso. Especial atencion a la lista de mensajes (`List<Message>` de Spring AI — puede que Jackson necesite `@JsonSubTypes` para deserializar polimorficamente, o un DTO propio del proyecto en lugar del tipo de Spring AI). Aplicar regla codigo real de S22 (`jar tf`, `unzip -p`, `-sources.jar`) sobre `Message`, `AssistantMessage`, `UserMessage`, `ToolResponseMessage` para decidir con evidencia.

3. **`RunStateRepository` — interfaz vs directamente clase concreta con Redis**. La deuda 18 introduce Redis. Se puede ir directo a clase concreta (YAGNI puro) o dejar interfaz por si en el futuro se prueban tests con implementacion en memoria. Discutir. Tender a YAGNI salvo argumento fuerte.

4. **Paquete `hitl` como nuevo top-level dentro del proyecto**. Comparar con estructura actual (`dto`, `config`, `agent`, `exception`, `controller`). Justificacion: HITL cruza varias capas (dominio del `AgentRunSession`, repository, properties, comportamiento en el bucle) — reagrupar por feature vertical tiene sentido. Alternativa: dispersar cada pieza en su paquete horizontal. Discutir.

### Elementos a discutir antes de teclear (si se cierra deuda 17 en S26-B)

1. **Version de `spring-boot-starter-data-redis`**. Vendra por el BOM de Spring Boot 4.1, no hay que fijarla explicitamente.

2. **Property exacta para apagar Redis en tests**. En `document-analyzer-ai` verificar cual se aplica: puede ser `spring.data.redis.repositories.enabled: false`, o `spring.autoconfigure.exclude: org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration`, o combinacion. No dar por hecho — verificar antes.

3. **`./mvnw test` sin Docker levantado**. Verificacion critica: los 9 tests siguen 9/9 verde sin `docker compose up` del Redis.

### Orden propuesto para S26-B

1. **Verificar hash del ultimo commit** con `git log -1 --oneline` (esperado: `4c8057d`) y `git status` (esperado: clean, sincronizado con origin).

2. **Decision temprana**: `HitlProperties` en S26-B o en S26-C. 5 min de discusion, decidir.

3. **Bloque diseno en pizarra**:
   - `AgentRunSession`: shape, tipos, serializacion (aplicando regla codigo real de S22 sobre Spring AI `Message`).
   - `RunStateRepository`: interfaz o directo, metodos.
   - Paquete `hitl` (o alternativa).
   - (Si se decidio incluir en S26-B) `HitlProperties`: shape, prefix, activacion.

4. **Guardar diseno en `S26-B-diseno-estado-persistido.md`** en el repo de notas. Cero codigo en este bloque.

5. **(Opcional) Cierre de deuda 17** si (a) el diseno justifica anadir la dependencia ya, y (b) queda tiempo y cabeza:
   - Anadir `spring-boot-starter-data-redis` al `pom.xml`.
   - Ver `application-test.yml` de `document-analyzer-ai` como referencia si aplica.
   - Anadir el override en `application-test.yml` del `erp-purchasing-agent`.
   - `./mvnw test` **sin Docker levantado del Redis** — verificar 9/9 verde.
   - `git status` doble, commit, push. Mensaje bilingue tipo `chore(deps): add spring-boot-starter-data-redis and disable in test profile`.

6. **Cierre**: notas de sesion S26-B, prompt de arranque S26-C.

**No haremos en S26-B** (van a partir de S26-C):

- Codigo del `RunStateRepository`.
- Codigo del `AgentRunSession`.
- Cambios en `AgentRunResult` (nuevo `status`, campos opcionales para pausa).
- Cambios en `ReActAgent` (break del bucle en HITL).
- Endpoint `POST /agent/run/{runId}/approve`.
- README.md profesional — mas adelante.
- OpenTelemetry / Micrometer — Fase 5.

## Estado esperado del entorno al arrancar S26-B

- `~/proyectos/erp-purchasing-agent/` sincronizado con `origin/main`. Working tree clean. `main` en `4c8057d`. `docker-compose.yml` presente con servicio Redis. Constructor de `ReActAgent` con 6 parametros. 9 tests verdes segun ultima verificacion (S25).
- `~/proyectos/erp-mcp-server/` **NO hace falta arrancarlo** para S26-B.
- `~/proyectos/document-analyzer-ai/` disponible como referencia de patron Redis (no se toca; si se cierra la deuda 17, hay que ver su `application-test.yml`).
- Contenedor `erp-purchasing-redis` puede estar arriba o abajo — irrelevante para el diseno en pizarra. Si se cierra la deuda 17, verificar tests **con el contenedor bajado**.
- Variable de entorno `ANTHROPIC_API_KEY` **no necesaria**.
- Docker daemon no imprescindible para el bloque diseno; necesario si se cierra la deuda 17 (para levantar/bajar Redis segun verificacion).

## Instrucciones para arrancar chat nuevo

Esperar mi primer mensaje. Al recibirlo, checklist estandar:

1. Como estoy de energia y cabeza.
2. Confirmar scope: seguimos con `erp-purchasing-agent`, sesion S26-B — diseno detallado en pizarra del estado persistido de HITL (`AgentRunSession` + `RunStateRepository` + posible `HitlProperties`) y posible cierre de deuda 17.

Ya no hace falta preguntar por Coditramuntana en la checklist. Si hay novedad, Tole la contara sin que se le pregunte.

**No hacer**:

- No resumir este prompt al arrancar.
- No re-explicar conceptos ya asentados de S19-S26-A (bucle explicito, `ToolCallingManager`, `hasToolCalls`, `AnthropicChatOptions`, `ProblemDetail`, BDDMockito, `MockitoExtension`, `Clock` inyectado, `Usage` interface, shape de `ChatResponse`, helpers `buildResponseWithoutToolCalls` y `buildResponseWithToolCalls`, refactor a `AgentRunResult`, snake_case global en Jackson, mapper en controller, shape de `ToolCall` / `ToolExecutionResult` / `ToolResponseMessage`, `Clock.fixed` vs mock, comportamiento de `willReturn` encadenado, `@WebMvcTest`, `@MockitoBean`, `isUnprocessableContent`, serializacion de enum en JSON, `within` de AssertJ, cascada MCP en `contextLoads`, perfil `test` con `application-test.yml`, `@ActiveProfiles`, HTTP 202 Accepted, HTTP 410 Gone, Strategy pattern, sintaxis de records, `@ConfigurationProperties`, `@ConfigurationPropertiesScan`, convencion `XxxProperties`, orden colaboradores-primero-properties-al-final, guardar record completo vs desempaquetar, patron `docker exec ... redis-cli` para pinguear Redis en contenedor).

- No pegar codigo completo por adelantado (Socrates puro).
- No proponer Claude Code para tests.
- No **implementar** `RunStateRepository`, `AgentRunSession`, `HitlProperties` ni el break del bucle en S26-B. Solo diseno en pizarra (y posible cierre de deuda 17 si el diseno lo justifica).
- No mezclar diseno en pizarra y (si se hace) commit de deuda 17 en un mismo bloque de discusion sin cierre limpio del diseno primero.

**Si hacer**:

- Verificar hash del commit con `git log -1 --oneline` al arrancar (esperado `4c8057d`).
- Decision temprana en 5 min sobre `HitlProperties` en S26-B o S26-C.
- Aplicar regla lockeada en S25 (refactor incremental con verificacion) si se cierra deuda 17: anadir dependencia, verificar compila; anadir override, correr tests, verificar 9/9.
- Aplicar regla lockeada en S19 (UNA decision por mensaje) — con vigilancia extra: el diseno en pizarra tiene muchas piezas moviendose (shape, tipos, serializacion, paquete, properties). Adelantarse tiene mas coste. Recordar al principio.
- Aplicar regla codigo real de S22 (`jar tf`, `unzip -p`, `-sources.jar`) para Spring AI `Message` y compania cuando se discuta el campo `conversationHistory` del `AgentRunSession`.
- Aplicar regla lockeada en S20 si la sesion pasa de 3h.
- `git status` doble antes de commit (si hay commit).
- Recordar warning menor de Mockito self-attaching (deuda tecnica 11, no bloquea).
- Aplicar regla reforzada en S26-A: no cerrar deudas antes de que la dependencia funcional exista. La deuda 17 solo se cierra en S26-B si se anade `spring-boot-starter-data-redis` al `pom.xml` en el mismo commit.

## Deuda tecnica activa (informativa, S26-A saldo la deuda 18)

Actualizada al cierre de S26-A. Las tachadas estan cerradas.

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
17. **Reformulada en S26-A, objetivo de S26-B (condicional)**: cuando se anada `spring-boot-starter-data-redis` al `pom.xml` (probablemente en S26-B o S26-C), apagar el autoconfigurador Redis en `application-test.yml` en el mismo commit. Verificar que `./mvnw test` sigue 9/9 verde sin Docker levantado.
18. ~~Redis debe anadirse al `docker-compose` del `erp-purchasing-agent`~~. **SALDADA S26-A** (`4c8057d`).

## Nota lateral registrada en S26-A: ElastiCache como posible modulo del roadmap AWS

Charla breve en S26-A: ElastiCache no es dificil conceptualmente, pero cuesta euros reales tenerlo levantado (no cubierto por Free Tier; nodo mas barato `cache.t4g.micro` ~9-12 USD/mes si se deja encendido). Se puede provisionar con Terraform en un rato (encaja bien con el roadmap AWS del otro chat) y luego destruir. El aprendizaje real: security groups, subnet groups, cifrado en transito, cluster mode vs single-node, failover. Encaja natural como modulo del roadmap AWS mas adelante, no aqui. Anotado.

## `redis-tools` en el host de Tole (nota logistica)

En S26-A, `redis-cli` no estaba instalado en el host. Intento de `sudo apt install redis-tools` colgo. Se opto por `docker exec erp-purchasing-redis redis-cli ping` como alternativa — patron habitual con Docker. Si en algun momento hace falta usar `redis-cli` frecuentemente desde el host, valorar instalarlo (verificar que `apt` no este bloqueado por unattended-upgrades o similar). No bloquea.

## Estado candidaturas (informativo, no forzar el tema)

Al cierre de S26-A:

- **Coditramuntana**: sin novedad reportada por Tole al arrancar S26-A (regla del prompt de arranque: ya no se pregunta). Sin accion planificada.
- **Otras candidaturas**: sin cambios reportados.

## Sesion de re-lectura guiada (informativa)

Sigue pendiente. Prompt en `PromptArranque-ReLecturaGuiada.md`. Tema propuesto: MapStruct o Testing (segun energia). No es S26-B.

## Alternativas si Tole prefiere cambiar de eje ese dia

- **AWS Sesion 12-E** (Route Tables privadas + RDS auto-start, chat separado).
- Sesion de re-lectura guiada (MapStruct o Testing).
- Deuda tecnica arco Redis: bug `LlmLoggingAdvisor` × `MessageChatMemoryAdvisor` (45-60 min).
- Solo el diseno del `AgentRunSession` (parte del bloque), sin llegar al `RunStateRepository` ni al paquete `hitl` — deja S26-B partida y el resto para el siguiente hueco. Util si Tole llega justo de tiempo o de cabeza.
