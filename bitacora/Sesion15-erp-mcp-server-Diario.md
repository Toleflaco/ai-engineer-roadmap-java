# Sesión 15 — erp-mcp-server: Persistencia PostgreSQL con Flyway + Dockerfile

**Fecha**: 2026-08-05
**Duración**: dos ventanas (mañana ~1h 20min + tarde ~3h)
**Fase**: 2 — AI Engineer roadmap, proyecto `erp-mcp-server`
**Bloque**: S15 — infraestructura (Dockerfile + migración PostgreSQL + CI/CD + README)
**Cobertura de la sesión**: docker-compose para PostgreSQL, Flyway con V1 (schema) y V2 (seed), Dockerfile multi-stage con layered jar. **Pendiente para próximas subsesiones**: integración del servicio `app` al docker-compose, CI/CD con GitHub Actions, README.

Cierre con 3 commits en `main` empujados a `origin`:
- `6134d33` chore: switch persistence to PostgreSQL with Flyway
- `6e26f79` feat: add Flyway migrations V1 schema and V2 seed data
- `2514728` feat: containerize app with multi-stage Dockerfile and layered jar

---

## Contexto de arranque

Sesión previa (14-cont-3) cerró la state machine completa de `PurchaseOrder` con `receivePurchaseOrder` y encapsulación de mutaciones de stock en `Product`. S15 abre bloque de infraestructura, cambio de tema respecto al dominio de las 4 sesiones anteriores.

Reloj muy fragmentado: **ventana de mañana 05:08–06:30** (empezar a trabajar corta la sesión), **ventana de tarde 16:10–~19:15** con retomas del staging conservado entre shells.

Decisión de orden al arrancar S15: **PostgreSQL → Dockerfile → CI/CD → README**. Motivo: cada paso alimenta al siguiente. Empaquetar con H2 antes de migrar a Postgres es rehacer. CI/CD antes de Dockerfile es CI incompleto. README al final es descriptivo, no aspiracional.

Decisión sobre sesión puente de Redis en `document-analyzer-ai`: **NO intercalarla hoy**. Se decide entre S15 y S16, no antes.

Regla operacional cambiada en esta sesión (ventana tarde, por presión de reloj + inglés): **mensajes de commit los redacto Claude, tú revisas minuciosamente antes de `git commit`**. No aplica a `@Tool`/`@ToolParam`/DTOs, que sigues escribiendo tú.

---

## Bloque S15-part-1 (mañana): Postgres arriba y schema versionado

### Docker Desktop apagado al arrancar

Ventana de mañana empezó con Docker Desktop apagado. Primera lección de scope: verificar el tooling básico antes de proponer flujo del día. Diagnóstico rápido con `which docker` / `ls /var/run/docker.sock`. Solución de 1 minuto (Docker Desktop desde el system tray de Windows).

### Postgres 14 nativo bloqueando el puerto 5432

Con Docker arriba, `docker ps` limpio pero `ss -tlnp | grep 5432` sí muestra proceso escuchando. Diagnóstico con `sudo ss -tlnp | grep 5432` reveló Postgres 14 instalado nativo en WSL bajo systemd, autoarrancado. Bases `inventario` y `task_manager` heredadas de trabajo en Fase 1. Datos preservados en `/var/lib/postgresql/14/main/`.

Solución:
```
sudo systemctl stop postgresql
sudo systemctl disable postgresql
```

`stop` para la sesión actual, `disable` evita que arranque solo en próximos WSL. No desinstalado: revive con `enable && start` si algún día se necesita.

**Deuda operacional lockeada**: verificar puerto libre (`sudo ss -tlnp | grep <puerto>`) antes de arrancar cualquier servicio con puerto expuesto. Aviso preventivo para próximos servicios en Kubernetes, agente ReAct de Fase 3, etc.

### docker-compose.yml para PostgreSQL

Decisiones al arrancar (todas empíricamente coherentes con roadmap):
- Servicio `postgres` (no `db`, no `erp-postgres` en el nombre lógico) — más legible, coincide con el motor.
- `container_name: erp-postgres` para no chocar con `tmm-postgres` de `task-manager-microservices`.
- Imagen: `postgres:17-alpine` (17 estable, alpine reduce a ~90MB).
- Usuario / DB / password: `erp` / `erp` / `erp` en dev.
- Volumen nombrado `erp-postgres-data`.
- Healthcheck con `pg_isready -U erp -d erp`.
- `restart: unless-stopped`.
- Sin `version:` top-level (Compose v5 lo ignora, deprecado en v3+).

**Bug recurrente detectado — Copia mecánica de literales al parametrizar**: al copiar el healthcheck del compose de `task-manager-api`, se olvidó cambiar `pg_isready -U task_manager_app -d task_manager`. Segundo intento: se sustituyó por `-U erp-postgres_app -d erp-postgres` (inventado, ni siquiera coincide con el usuario/base declarados arriba en el mismo yml).

**Patrón general**: al parametrizar un template copiado, cada literal se sustituye leyendo primero qué representa el flag/campo original, no por analogía fonética con el proyecto nuevo. Extensión de la regla "copia mecánica al escribir método análogo" ya lockeada en 14-cont-3.

Al final, `docker compose up -d` limpio: contenedor `Up 17 seconds (healthy)`. Verificado con `docker exec erp-postgres psql -U erp -d erp -c "\l"`.

### Ajustes en pom.xml y application.properties

**Spring Boot 4.x modularizó los starters.** Autoconfig de Flyway ya no viene con `flyway-core` a secas: requiere `spring-boot-starter-flyway` explícito. Dependencias añadidas al pom:

1. `org.springframework.boot:spring-boot-starter-flyway` — starter (activa autoconfig).
2. `org.flywaydb:flyway-database-postgresql` — dialecto Postgres (obligatorio desde Flyway 10+).
3. `org.postgresql:postgresql` con `scope=runtime` — driver JDBC.

Quitadas: `com.h2database:h2` y `com.h2database:spring-boot-h2console`.

`application.properties`:
- Datasource cambia a `jdbc:postgresql://localhost:5432/erp`.
- `spring.jpa.hibernate.ddl-auto=validate` (Flyway gestiona schema, Hibernate valida coincidencia).
- Sin `spring.jpa.defer-datasource-initialization` (deja de tener sentido con Flyway al mando).
- Sin consola H2.
- Se intentó declarar `spring.jpa.properties.hibernate.dialect=PostgreSQLDialect` explícito, pero Hibernate emite HHH90000025 recomendando quitarlo (autodetecta desde el driver). Finalmente eliminado.

### Test scaffolding borrado

`ErpMcpServerApplicationTests.java` era el `@SpringBootTest` vacío con solo `contextLoads()`. Con `ddl-auto=validate` + PostgreSQL, ese test requeriría Postgres vivo para cargar contexto y no tiene assertions. Borrado, tests reales llegarán en Fase 3.

### Corte de mañana con staging armado

Commit no ejecutado (los mensajes bilingües comían demasiado tiempo antes de arrancar a trabajar). Staging conservado entre shells (Git preserva staging entre sesiones). Salida a las 06:30 con Postgres corriendo, 4 ficheros en staging (`docker-compose.yml`, `pom.xml`, `application.properties`, delete de `ErpMcpServerApplicationTests.java`).

---

## Bloque S15-part-2 (tarde): commit de infra, Flyway V1/V2, Dockerfile

### Retomada y commit de infra

Al retomar a las 16:10: `git status` confirma staging intacto. `docker compose ps` muestra `erp-postgres` healthy en 13 seg (Docker Desktop reinició el contenedor con el volumen).

Cambio de regla operacional: mensajes de commit los redacto Claude. Tú revisas. Commit `6134d33` con mensaje bilingüe (`chore: switch persistence to PostgreSQL with Flyway`). Tipo `chore` porque es infra sin nueva feature de dominio.

### V1__init_schema.sql

Decisiones transversales sobre schema (todas lockeadas):

- **PKs con `BIGSERIAL`**. Encaje directo con `@GeneratedValue(strategy = IDENTITY)` que ya tienen las entidades.
- **Sin `ON DELETE CASCADE`**. JPA gestiona ciclo de vida vía `orphanRemoval`. Cascada SQL enmascararía bugs.
- **`TIMESTAMPTZ` para timestamps de auditoría**. Mapea a `OffsetDateTime` sin conversión. Coherente con `@PrePersist` que usa `OffsetDateTime.now(ZoneOffset.UTC)`.
- **NOT NULL estricto**. `ddl-auto=validate` no compara nullability, pero la BD es la última línea de defensa contra bugs futuros de inserción manual.
- **`VARCHAR(255)` para texto general, `VARCHAR(20)` para enums**. Los enums son discriminadores cortos y merecen documentación de contrato.
- **Sin índices en FKs**. En PostgreSQL las FKs **no crean índice automático** (a diferencia de MySQL/InnoDB). Con seed <100 filas, sequential scan es imperceptible. Añadir índices en `V3__add_fk_indexes.sql` cuando aparezca coste perceptible. Regla operacional: cada FK viene con la pregunta "¿voy a JOINear frecuentemente por aquí?". Si sí, índice; si no lo sé, aplazar.
- **`UNIQUE` explícito en `invoices.purchase_order_id`**. Hibernate no crea constraint UNIQUE automática en `@OneToOne`; sin ella, dos facturas podrían apuntar a la misma orden y romper la semántica.
- **`unit_price NUMERIC(12, 2)`**. Cierra el TODO inline `// TODO(sesion-15): revisar precision/scale al migrar a PostgreSQL`. Coherente con `Invoice.amount` y `PurchaseOrderLine.unit_price_at_order`.

Cinco tablas creadas: `suppliers`, `products`, `purchase_orders`, `purchase_order_lines`, `invoices`. Verificado con `\dt` (6 relaciones: las 5 más `flyway_schema_history`).

### V2__seed_dev_data.sql

**Bug de sintaxis H2 detectado**: el `data.sql` original usaba `ALTER TABLE <tabla> ALTER COLUMN id RESTART WITH 100` (sintaxis H2). En PostgreSQL el reset de sequence se hace sobre la sequence directamente: `ALTER SEQUENCE <tabla>_id_seq RESTART WITH 100`.

Nombres de sequence generados automáticos: `<tabla>_<columna>_seq`. Confirmado empíricamente con `\d products` que muestra `nextval('products_id_seq'::regclass)`.

Sustituciones al portar `data.sql` a `V2__seed_dev_data.sql`:
- 5 líneas `ALTER TABLE ... RESTART WITH 100` → `ALTER SEQUENCE <tabla>_id_seq RESTART WITH 100`.
- Sin más cambios: los `INSERT`, los `TIMESTAMP WITH TIME ZONE '...'`, los valores de enum como string, y los IDs explícitos son compatibles Postgres directos.

Seed inalterado en cuanto a datos: 7 suppliers, 19 products, 5 purchase orders, 13 lines, 2 invoices.

Ubicación: `src/main/resources/db/migration/V2__seed_dev_data.sql`. Convención Flyway obligatoria (`V<version>__<descripción>.sql` con dos underscores).

`data.sql` original borrado (Spring Boot no lo ejecuta con Postgres por defecto salvo `spring.sql.init.mode=always`, pero mantenerlo confunde).

Verificado empíricamente: Flyway aplicó V1 y V2 en 52ms total. Row counts exactos: 7/19/5/13/2. Sequence `suppliers_id_seq.last_value = 100`. 13 tools registradas (7 lectura + 3 escritura + 3 transición; el prompt de continuación tenía "10" mal contado).

Commit `6e26f79` con mensaje bilingüe (`feat: add Flyway migrations V1 schema and V2 seed data`). Cierre inline TODO de `Product.unitPrice` mencionado en el mensaje **y borrado del código** en el mismo commit (opción 1: mejor coherencia commit ↔ código).

### Dockerfile multi-stage con layered jar

Decisiones locked:

- **Multi-stage**: stage 1 con JDK completo compila, stage 2 con solo JRE ejecuta. Imagen final ~157MB comprimido / ~526MB descomprimido con capas base. Patrón esperado en CV banca/consultoría.
- **Base**: `eclipse-temurin:21-jdk-jammy` (build) + `eclipse-temurin:21-jre-jammy` (runtime). No Alpine (compatibilidad con `glibc` y SDKs cloud es importante para el CV).
- **Layered jar** habilitado en `pom.xml` con `<layers><enabled>true</enabled></layers>` dentro del plugin. Extrae 4 layers separadas: `dependencies` (63.8MB), `spring-boot-loader` (696kB), `snapshot-dependencies` (4.1kB), `application` (262kB). Cambios de código solo invalidan la última capa; las otras 3 se reutilizan cacheadas.
- **Usuario no-root**: `spring:spring` creado con `groupadd/useradd --system`, `USER spring` antes de los COPY. `--chown=spring:spring` en los COPY para que los ficheros no lleguen como `root:root`.
- **`ENTRYPOINT` en exec form**: `["java", "org.springframework.boot.loader.launch.JarLauncher"]`. JVM como PID 1 recibe SIGTERM directo, graceful shutdown funciona. Nota: en Boot 3.2+ el package pasó a `launch.JarLauncher` (antes era solo `loader.JarLauncher`).
- **`EXPOSE 8080`** declarativo (metadato, no publicación real).
- **HEALTHCHECK Dockerfile**: NO por ahora (requeriría `spring-boot-starter-actuator`, scope de subpaso siguiente).
- **Integración compose (D6)**: NO hoy. Primero Dockerfile aislado + `docker run` verifica que fabrica una imagen que arranca; luego integración al compose en subpaso siguiente.

Cadena de bugs corregidos durante escritura:

1. **`spring-boot-maven-plugin` sin groupId explícito rompe en Boot 4.x**. Warning: `'build.plugins.plugin.version' for org.apache.maven.plugins:spring-boot-maven-plugin is missing`. Maven defaulteaba a `org.apache.maven.plugins`. En 3.x el pluginManagement del parent hacía el match sin groupId; en 4.x con la modularización dejó de funcionar. Solución: añadir `<groupId>org.springframework.boot</groupId>` explícito.

2. **`extract` sin `--destination` crea subdirectorio con nombre del jar**. Los COPY buscaban `/app/target/extracted/dependencies/` y no encontraban nada porque el extract lo dejaba en `/app/target/extracted/erp-mcp-server-0.0.1-SNAPSHOT/dependencies/`. Solución: `--destination target/extracted` explícito.

`.dockerignore` con exclusión de `target/`, `.git/`, `.idea/`, `.vscode/`, `*.iml`, `*.md`, `Dockerfile`, `.dockerignore`, `docker-compose.yml`. Filosofía B (build dentro del contenedor) hace irrelevante el `!target/*.jar`, pero se deja como red de seguridad.

Verificado empíricamente:
- `docker build -t erp-mcp-server:0.0.1 .` OK.
- `docker run --rm -d --name erp-mcp-server --network erp-mcp-server_default -e SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/erp -p 8080:8080 erp-mcp-server:0.0.1` arranca.
- Log muestra `Database JDBC URL [jdbc:postgresql://postgres:5432/erp]` — evidencia del override por env var.
- Flyway ve V1 y V2 aplicadas, no reaplica.
- 13 tools respondiendo vía mcp-inspector: `getSupplier(1)`, `getProduct(2)`, `listPurchaseOrdersByStatus("RECEIVED")`, `getPurchaseOrder(1)` — todas OK.
- Escritura: `createSupplier("Test Supplier S15", "test@s15.local")` devuelve `id=100` (sequence tras `RESTART WITH 100`), persistencia confirmada con `getSupplier(100)` retornando el DTO con `products: []`.

Commit `2514728` con mensaje bilingüe (`feat: containerize app with multi-stage Dockerfile and layered jar`). Se coló whitespace irrelevante en `Product.java` en el staging (línea vacía con y sin espacio trailing por autoformat del IDE). Detectado post-commit con `git show --stat`. No se rehace: cambio semánticamente nulo.

### Concepto Dockerfile vs docker-compose (explicación pedagógica)

Momento de confusión durante el bloque tarde: ¿cuál es la diferencia entre Dockerfile y docker-compose?

Aclaración lockeada:
- **Dockerfile = receta para fabricar UNA imagen**. Uno por proyecto que compilas tú. `docker build -t X .` produce imagen.
- **docker-compose.yml = orquestador que arranca VARIOS contenedores**. Coordina red, orden, config. No fabrica imágenes por defecto, las descarga de Docker Hub (`image:`) o delega la fabricación al Dockerfile (`build: .`).
- **Postgres** en el compose actual: `image: postgres:17-alpine` — no hay Dockerfile porque la imagen es de la comunidad.
- **Tu app MCP server**: `Dockerfile` obligatorio porque nadie ha publicado `erp-mcp-server` en Docker Hub, tú la fabricas.

### Concepto SPRING_DATASOURCE_URL como override (explicación pedagógica)

Segundo momento de confusión, ya cerca del final: ¿qué hace `-e SPRING_DATASOURCE_URL=...` en el `docker run`?

Aclaración lockeada:
- Spring Boot ofrece **externalized configuration**: cualquier propiedad de `application.properties` puede ser sobrescrita por una env var.
- Conversión de nombre: puntos → underscore, mayúsculas. `spring.datasource.url` → `SPRING_DATASOURCE_URL`.
- Precedencia: env var > `application.properties`.
- Utilidad: una misma imagen Docker vale para local, docker-compose, Kubernetes, prod. La config específica del entorno viene por env var. Es el patrón "twelve-factor app".

Evidencia empírica: log de la app mostró `Database JDBC URL [jdbc:postgresql://postgres:5432/erp]` aunque el properties dijera `localhost`.

---

## Estado técnico al cierre

### Repo `erp-mcp-server`

Rama `main`, `origin` sincronizado. Último commit `2514728`.

Estructura nueva:
- `docker-compose.yml` en raíz (solo Postgres por ahora).
- `Dockerfile` en raíz (multi-stage, layered jar, no-root).
- `.dockerignore` en raíz.
- `src/main/resources/db/migration/V1__init_schema.sql` (5 tablas de dominio).
- `src/main/resources/db/migration/V2__seed_dev_data.sql` (seed portado desde H2).
- `src/main/resources/application.properties` limpio para Postgres.
- `pom.xml` con starter Flyway, driver Postgres, dialecto Postgres, plugin config layered.

Estructura eliminada:
- `src/test/java/dev/toleflaco/erpmcpserver/ErpMcpServerApplicationTests.java`.
- `src/main/resources/data.sql` (portado a V2).
- Dependencias H2 en `pom.xml`.
- Consola H2 en `application.properties`.

### Verificaciones empíricas cerradas

- 6 tablas creadas por V1 (5 dominio + `flyway_schema_history`).
- `\d products` muestra `unit_price numeric(12,2) NOT NULL`, PK con índice btree, FK saliente y entrante.
- Row counts post-V2: 7 suppliers, 19 products, 5 purchase orders, 13 lines, 2 invoices.
- `suppliers_id_seq.last_value = 100` tras seed (avanzada a 101 tras el test de escritura).
- App arrancando desde IntelliJ y desde contenedor.
- 13 tools respondiendo, incluyendo escritura con id de sequence correcto.

### BD residuo del test

Hay un `Test Supplier S15` con id=100 persistido en el volumen Postgres. Para partir de estado limpio: `docker compose down -v && docker compose up -d`. Sin urgencia.

---

## Deudas técnicas nuevas de esta sesión

Para README S15 (todavía por escribir en subsesión posterior):

- **Índices en FKs (V3__add_fk_indexes.sql)**. Diferidos hasta perfilado con datos reales.
- **`spring-boot-starter-actuator`** para HEALTHCHECK del Dockerfile (aplazado en D5).
- **Integración `app` al docker-compose.yml** (aplazado en D6): añadir servicio con `build: .`, `depends_on: postgres: condition: service_healthy`, `environment: SPRING_DATASOURCE_URL`.

Heredadas de 14-cont-3:
- Recepción parcial de pedidos (`quantityReceived` + `PARTIALLY_RECEIVED`).
- Setters públicos en `PurchaseOrderLine`.

Heredadas de bloque previo, sin cambios:
- Optimización con `findAllById` batch en `createPurchaseOrder`.
- Verificación empírica de nested records en Spring AI 2.0.
- Evaluación `OffsetDateTime` → `Instant`.
- Bean Validation con handler centralizado.
- Regex de email pragmático.
- N+1 sobre `suppliers` en `ProductQueryTools`.
- Serialización `BigDecimal` sin padding en totales calculados en memoria.
- Case sensitivity en derived queries.
- Refactor `groupId` `com.mtole` → `dev.toleflaco` en `task-manager-api` y `task-manager-microservices`.
- `GlobalExceptionHandler` con `NoResourceFoundException` en `document-analyzer-ai`.
- Redis en `document-analyzer-ai` (candidato a sesión puente entre S15 y S16).
- Migración de Claude Code al native installer.
- `apt upgrade` pendiente en Ubuntu WSL.

### Warnings cosméticos observados en el arranque, no bloquean

- `SyncMcpToolProvider: No tool methods found in the provided tool objects: []` — Spring AI 2.0 corre dos scanners (annotation-scanner nuevo + MethodToolCallbackProvider antiguo). Uno no encuentra tools por su vía. Las 13 se registran por el otro.
- `BeanPostProcessorChecker` sobre `McpServerAnnotationScannerAutoConfiguration`, `serverAnnotatedBeanRegistry`, `stringOrNumberMigrationVersionConverter`, etc. — coreografía de arranque de Spring AI + Flyway. Cosmético.

---

## Deudas operacionales lockeadas nuevas

### Copia mecánica de literales al parametrizar templates

Extensión de la regla "copia mecánica al escribir método análogo" ya lockeada. **Al parametrizar un template copiado, cada literal se sustituye leyendo primero qué representa el flag/campo original, no por analogía fonética con el proyecto nuevo**.

Bug del healthcheck del docker-compose:
- Primer intento: heredó `pg_isready -U task_manager_app -d task_manager` sin cambios.
- Segundo intento: sustituyó por `-U erp-postgres_app -d erp-postgres` — inventado por analogía, incoherente con el `POSTGRES_USER=erp` / `POSTGRES_DB=erp` declarados en el mismo yml.
- Tercer intento (correcto): `-U erp -d erp` leyendo qué representa cada flag.

### Verificar puerto libre antes de arrancar servicio con puerto expuesto

Al arrancar cualquier servicio con puerto expuesto:
```
sudo ss -tlnp | grep <puerto> || echo "puerto libre"
```

Aviso preventivo para Kubernetes en S16-S17, agente ReAct en S18-S20, cualquier otro servicio de futuros bloques.

### En PostgreSQL, FK no crea índice automático

Regla: cada declaración de FK viene con la pregunta "¿voy a JOINear frecuentemente por aquí?".
- Si sí: `CREATE INDEX idx_<tabla>_<columna> ON <tabla>(<columna>)`.
- Si no lo sé: aplazar hasta que aparezca coste perceptible con datos reales.

En MySQL/InnoDB el índice se crea implícito; el reflejo condicionado engaña al llegar a Postgres.

### Cambio operacional en la regla de commits

Por presión de reloj + inglés: **mensajes de commit los redacta Claude, Tole revisa minuciosamente antes de `git commit`**. Aplica desde 2026-08-05.

NO se aplica a:
- `@Tool` y `@ToolParam` descriptions.
- Nombres, campos y tipos de DTOs.
- Nombres de métodos de dominio.

Estos siguen siendo escritos por Tole primero, discusión después.

### Verificar `git status` DOS veces (antes Y después de `git add`)

Regla ya lockeada, con caso empírico nuevo: en el commit `2514728`, el primer `git status` mostraba `Product.java` en staging **antes** del `git add` planeado. No se detectó, siguió el flujo, `Product.java` entró al commit. Diagnóstico post-hoc: cambio era whitespace irrelevante, sin acción correctiva.

Aprendizaje: cuando el primer `git status` muestra más ficheros de los esperados, **parar y diagnosticar**, no seguir con `git add` planeado.

### Spring Boot 4.x modulariza y rompe herencias

Consecuencia directa de la modularización de Boot 4.x que se ha manifestado tres veces en la sesión:
1. Flyway autoconfig requiere `spring-boot-starter-flyway` explícito (no bastaba con `flyway-core`).
2. `spring-boot-maven-plugin` requiere `<groupId>org.springframework.boot</groupId>` explícito.
3. `flyway-database-postgresql` para dialecto Postgres.

Regla: **en Spring Boot 4.x, ante cualquier warning "not found", "no bean of type ...Autoconfiguration", "plugin missing"**, primer sospechoso es que **falta el starter modular** o **falta el groupId explícito**. Diferencia clara con hábitos de Boot 3.x.

---

## Ⓥ Frases ⭐⭐⭐ para entrevistas

- **Flyway 10+ requiere dialecto separado**: "en Spring Boot 3.2+ con Flyway 10, hay que añadir `flyway-database-postgresql` explícito o Flyway falla en arranque con 'no database plugin found'".
- **Spring Boot 4.x modulariza starters**: "en Boot 4, la autoconfig de Flyway se movió a un módulo separado, hay que añadir `spring-boot-starter-flyway` explícito además de `flyway-core`; y `spring-boot-maven-plugin` necesita groupId declarado, el pluginManagement heredado no aplica ya".
- **`ddl-auto=validate` con Flyway al mando**: "cuando Flyway gestiona el schema, Hibernate va con `ddl-auto=validate` para que valide coincidencia entidad-schema pero no cree nada; es red de seguridad contra drift".
- **PostgreSQL no indexa FKs automático**: "en Postgres, PK y UNIQUE llevan índice implícito, pero las FKs no; hay que crearlos explícitos si vas a JOINear frecuentemente".
- **`ALTER SEQUENCE ... RESTART WITH N`**: "el equivalente en Postgres del `RESTART WITH` de H2 va sobre la sequence directamente, no sobre la columna: `ALTER SEQUENCE <tabla>_id_seq RESTART WITH 100`".
- **Multi-stage Dockerfile + layered jar**: "stage 1 con JDK completo compila el jar layered, stage 2 con solo JRE recibe las 4 capas separadas; cambios de código solo invalidan la capa `application`, las dependencias siguen cacheadas".
- **Exec form del ENTRYPOINT**: "siempre exec form `["java", "..."]`, no shell form; para que la JVM sea PID 1 y reciba SIGTERM directo del kernel, y el graceful shutdown de Spring funcione".
- **SPRING_ prefix como override**: "cualquier property de Spring es sobrescribible por env var con conversión `puntos→underscore` y mayúsculas; una imagen sirve para todos los entornos si la config va por env var".
- **`docker run --network` para simular compose sin compose**: "para probar una imagen contra la red que compose ya creó, `--network <proyecto>_default` + resolución por nombre de servicio".

---

## Próxima subsesión (S15-part-3)

Objetivo: **integración del servicio `app` al `docker-compose.yml`**.

Cuatro conceptos nuevos:
1. `build: .` en el servicio `app`.
2. `depends_on: postgres: condition: service_healthy` (aprovechando el healthcheck ya escrito).
3. `environment: SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/erp` en el compose.
4. Decisión: ¿publicar 8080 al host o mantener el servicio interno a la red?

Coste estimado: 30-45 min. Después vienen CI/CD (60-90 min) y README (45-60 min) como subsesiones aparte.

Alternativa: sesión puente Redis en `document-analyzer-ai` antes de continuar S15. Coste: media sesión. Decisión a tomar al arrancar la próxima.

---

**Estado emocional al cierre**: sesión larga (~4h totales entre las dos ventanas), avance significativo pero con muchas microdesviaciones diagnosticadas (Docker apagado, Postgres nativo, modularización Boot 4.x, `extract --destination`, whitespace de `Product.java`). Cada una consumió tiempo pero cada una produjo aprendizaje que queda lockeado. Corte en momento correcto: escribir integración compose bajo fatiga arrastraría bugs propios del scope operacional que se ven hoy en el retroanálisis.
