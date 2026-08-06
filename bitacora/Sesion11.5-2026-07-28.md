# Sesión 11.5 — Seed data H2 para erp-mcp-server

- **Fecha:** 2026-07-28
- **Fase:** 2 (Tool calling, agentes y MCP)
- **Duración:** ~1h, sesión corta intercalada entre Sesión 11 y Sesión 12
- **Estado del roadmap:** Sesión 11.5 completada. Las 5 tablas de `erp-mcp-server` tienen seed coherente y verificado en H2. Queda abierta la Sesión 12 (starter de Spring AI MCP Server).

## Contexto

Sesión corta, prevista ya en el cierre de la Sesión 11: las cinco entidades JPA quedaron completas pero sin datos. Objetivo único: cargar seed data coherente en H2 para que la Sesión 12 pueda arrancar con datos reales que las tools MCP puedan devolver, en lugar de una base vacía.

## Trabajo técnico

### data.sql

Se creó `src/main/resources/data.sql`, organizado en cinco bloques con `INSERT` explícitos por tabla, en el orden que respeta las dependencias de FK:

- **7 suppliers**: proveedores del dominio café/hostelería (café verde, tostadores, lácteos, envases, equipamiento de barista, siropes, té).
- **19 products**, repartidos entre los 7 suppliers. De ellos, 6 con `stock < min_stock` (pensando en una futura tool de detección de low stock) y 2 en rotura total (`stock = 0`).
- **5 purchase_orders**, una por cada valor del enum `PurchaseOrderStatus`: `RECEIVED` ×2, `SENT`, `DRAFT`, `CANCELLED`.
- **13 purchase_order_lines**, repartidas entre las 5 POs, con `unit_price_at_order` como snapshot histórico. En varias líneas el precio congelado difiere a propósito del `unit_price` actual del producto, para que la regla de dominio discutida en Sesión 11 (snapshot, no caché) sea visible en los datos y no solo en el código.
- **2 invoices**, una `PAID` y una `PENDING`, asociadas a las dos POs en estado `RECEIVED` mediante la relación `@OneToOne` con `unique` en `purchase_order_id`.

### Convención de IDs de seed

Cada bloque de tabla termina con:

```sql
ALTER TABLE suppliers
    ALTER COLUMN id RESTART WITH 100;
```

Los IDs del seed son explícitos y arrancan en 1. Sin este `ALTER TABLE`, la secuencia autoincremental de H2 seguiría desde el último valor insertado por el script (por ejemplo, 20 en `products`), lo cual funcionaría hoy pero es frágil: cualquier ampliación futura del seed movería el punto de arranque de la secuencia sin que nadie lo decidiera explícitamente. Fijar el reinicio en 100 en cada tabla, sin importar cuántas filas tenga el seed, deja un hueco deliberado y documentado entre IDs de seed e IDs de runtime.

### spring.jpa.defer-datasource-initialization

Con `ddl-auto=create-drop`, Spring Boot ejecuta por defecto `data.sql` antes de que Hibernate genere el schema, porque el flujo estándar de inicialización de datasource no espera a JPA. Contra tablas que aún no existen, el resultado es un `table not found` en el primer `INSERT`. Se añadió a `application.properties`:

```properties
spring.jpa.defer-datasource-initialization=true
```

Esta propiedad invierte el orden: Hibernate crea el schema primero, `data.sql` se ejecuta después. Es la propiedad correcta cuando el schema lo genera JPA (`ddl-auto`) y no un `schema.sql` propio.

## Verificación empírica

Arranque con `./mvnw spring-boot:run` y conexión a la consola H2 (`/h2-console`, JDBC URL `jdbc:h2:mem:erpdb`, user `sa`, sin password). Se lanzaron `SELECT` de conteo sobre las 5 tablas y se comprobó ausencia de huérfanos (FKs de `products`, `purchase_order_lines` e `invoices` resolviendo contra filas existentes). Verificación específica de las dos reglas de negocio codificadas en el seed:

```sql
SELECT * FROM products WHERE stock < min_stock;   -- 6 filas
SELECT * FROM products WHERE stock = 0;            -- 2 filas
SELECT status, count(*) FROM purchase_orders GROUP BY status;  -- 4 estados representados
```

Los tres resultados coincidieron con lo esperado por diseño del seed.

## Decisiones tomadas

- **Reinicio de secuencia a 100 tras cada bloque de seed**, en todas las tablas con ID autoincremental. Motivo: separar de forma explícita y visible el rango de IDs de seed (1–N) del rango de IDs que generará el runtime, evitando colisiones si el seed crece más adelante. Convención a repetir en cualquier `data.sql` futuro del roadmap.
- **`spring.jpa.defer-datasource-initialization=true` pasa a decisión locked** para todo proyecto JPA del roadmap que use seed vía `data.sql` bajo `ddl-auto=create-drop` (o `create`). Motivo: sin ella, el orden de inicialización por defecto de Spring Boot rompe el seed contra un schema generado por Hibernate.

## Notas de calibración

Sesión deliberadamente acotada a SQL puro, sin tocar código Java ni configuración MCP, para no mezclar el cambio de contexto de seed data con el arranque de Spring AI MCP Server previsto en Sesión 12. Decisión de secuenciación tomada ya en el cierre de la Sesión 11 y confirmada aquí.

## Commits generados

- `b342dc5` — `chore: add H2 seed data for development`

## Roadmap de Fase 2 actualizado

- Sesión 10 ✓: repo + esqueleto + 3 entidades + 2 commits.
- Sesión 11 ✓ (parcial): `PurchaseOrderLine`, `Invoice`, `@OneToMany`, encapsulación de agregado, ajustes de configuración, 4 commits.
- Sesión 11.5 ✓: `data.sql` con seed coherente (7 suppliers, 19 products, 5 POs, 13 líneas, 2 invoices), `defer-datasource-initialization`, verificación en consola H2, 1 commit.
- Sesión 12: Spring AI MCP Server starter + primera tool `getProduct(id)` + verificación con mcp-inspector.
- Sesiones 13-20: sin cambios respecto al plan de Sesión 10.

## Próximos pasos

- Sesión 12: añadir el starter de Spring AI MCP Server, primera tool de lectura sobre los datos ya sembrados, verificación con mcp-inspector.

## Deudas técnicas activas

Sin deudas nuevas. Heredadas de Sesión 11, sin cambios:

- `Product.unitPrice` sin `precision`/`scale` declarados. TODO inline en el fichero para revisar en Sesión 15 al migrar a PostgreSQL.
- Refactor del `groupId` de `com.mtole` a `dev.toleflaco` en `task-manager-api` y `task-manager-microservices` (prioridad media).
- Ampliar `GlobalExceptionHandler` con `NoResourceFoundException` en `document-analyzer-ai` (prioridad baja).
- Migración de Claude Code del instalador npm al native installer (prioridad baja).
- `apt upgrade` pendiente en Ubuntu WSL (prioridad baja).
- Deudas pedagógicas de Fase 1 (divergencias documentadas en el README del hub).
