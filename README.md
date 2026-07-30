# AI Engineer Roadmap · Java + Spring AI

Roadmap público de mi transición de Java backend a AI Engineer, con cuatro proyectos y bitácora en abierto.

---

## Sobre este repositorio

Soy Manuel Toledano, desarrollador Java backend. Después de reactivar mi carrera en desarrollo en 2026 con dos proyectos backend serios (monolito y microservicios sobre Spring Boot), he decidido apostar por la IA aplicada como siguiente paso lógico: no abandono Java, lo llevo al terreno donde el mercado está demandando perfiles. Este repositorio documenta ese camino en abierto.

Es un hub. No contiene código: enlaza a los cuatro repositorios de proyecto y aloja la bitácora de progreso y las decisiones arquitectónicas transversales.

## El roadmap en una tabla

| Fase | Contenido | Estado |
| --- | --- | --- |
| 0   | Prerrequisitos Java + Spring Boot sólido | Completada |
| 1   | Fundamentos LLM + Spring AI core | Completada |
| 2   | Tool calling, agentes y MCP | En curso |
| 3   | RAG empresarial con vector stores | Planificada |
| 4   | Observabilidad, testing y seguridad enterprise | Planificada |
| 5   | Cloud, despliegue y LLMOps sobre AWS | Planificada |
| 6   | Criterio arquitectónico y portfolio | Planificada |

Cada fase termina con un proyecto entregable en un repositorio independiente. La bitácora de sesiones registra el progreso día a día. La Fase 1 se cerró técnicamente en la Sesión 08.

## Los cuatro proyectos

1. **Analizador de CVs con Spring AI y Anthropic Claude** (Fase 1). Servicio Spring Boot con tres endpoints: chat con memoria conversacional (in-memory) segmentada por conversationId, análisis estructurado de CV en texto plano, y análisis multimodal de CV en PDF nativo. Observabilidad de latencia, tokens y coste estimado por llamada. Manejo de errores homogéneo con ProblemDetail RFC 7807. → [Completado con divergencias del plan original](https://github.com/Toleflaco/document-analyzer-ai)
2. **`erp-mcp-server`** (Fase 2). Servidor MCP sobre Spring Boot 4.1 y Spring AI 2.0 que expone tools de lectura y escritura sobre un dominio ERP (productos, proveedores, pedidos de compra, facturas), verificado end-to-end con mcp-inspector. Segundo repo de la fase, `erp-purchasing-agent` (agente ReAct consumidor de estas tools), pendiente de arrancar. → [En desarrollo](https://github.com/Toleflaco/erp-mcp-server)
3. **Knowledge base empresarial multi-tenant** (Fase 3). Sistema RAG con búsqueda híbrida, reranking, citación obligatoria y aislamiento estricto entre tenants. → *Planificado*
4. **AI Gateway multi-modelo en AWS** (Fase 5). Servicio en ECS Fargate con routing entre modelos, cache semántico, fallback automático y observabilidad de coste con Prometheus y Grafana. → *Planificado*

## Divergencias del plan original

El roadmap fusiona el Master AI Engineer de codeja.dev con un Roadmap Maestro personal. El plan de Fase 1 preveía análisis de contratos PDF, memoria conversacional persistente en Redis, y tres proveedores LLM intercambiables. Por prioridad de búsqueda activa de empleo, la Fase 1 se ha cerrado con divergencias: dominio CV en lugar de contratos, memoria in-memory en lugar de Redis, un solo proveedor (Anthropic Claude Sonnet 4.5) en lugar de tres. Los objetivos de aprendizaje de la fase (ChatClient, output estructurado, memoria conversacional, multimodal, manejo de errores, observabilidad básica) quedan cubiertos.

Estas divergencias se tratan como deuda pedagógica explícita, no como piezas terminadas. Plan post-empleo: completar los huecos originales de cada fase cuando desaparezca la presión de candidaturas.

## Cómo navegar el repositorio

- **`bitacora/`** — Una entrada por sesión de estudio, en orden cronológico. Actualizada hasta la [Sesión 13](bitacora/2026-07-30-sesion-13.md).
- **`decisions/`** — ADRs transversales que afectan a más de un proyecto (formato Michael Nygard). Aparecerá cuando exista el primero.
- **`CLAUDE.md`** — Instrucciones de trabajo para Claude Code en este repositorio.

## Contexto

- **Ubicación:** Cantabria (España). Trabajo remoto.
- **Perfil profesional:** [LinkedIn](https://www.linkedin.com/in/manueltoledano/).
- **Otros repositorios de referencia:** [task-manager-api](https://github.com/Toleflaco/task-manager-api) (monolito Spring Boot 4 con JWT, JPA, MongoDB, Testcontainers), [task-manager-microservices](https://github.com/Toleflaco/task-manager-microservices) (descomposición en microservicios con API Gateway, Resilience4j y Kafka).

---

*Última actualización: 2026-07-30*
