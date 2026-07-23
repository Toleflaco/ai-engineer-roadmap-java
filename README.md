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
| 1   | Fundamentos LLM + Spring AI core | En curso |
| 2   | Tool calling, agentes y MCP | Planificada |
| 3   | RAG empresarial con vector stores | Planificada |
| 4   | Observabilidad, testing y seguridad enterprise | Planificada |
| 5   | Cloud, despliegue y LLMOps sobre AWS | Planificada |
| 6   | Criterio arquitectónico y portfolio | Planificada |

Cada fase termina con un proyecto entregable en un repositorio independiente. La bitácora de sesiones registra el progreso día a día.

## Los cuatro proyectos

1. **API de análisis inteligente de documentos** (Fase 1). Servicio Spring Boot que analiza contratos PDF, extrae cláusulas estructuradas y permite consulta conversacional con memoria persistente. Tres proveedores LLM intercambiables. → *Próximo*
2. **Servidor MCP empresarial y agente consumidor** (Fase 2). Servidor MCP que expone funciones de un ERP simulado como herramientas de IA, más un agente Spring AI con bucle ReAct, guardrails de presupuesto y human-in-the-loop. → *Planificado*
3. **Knowledge base empresarial multi-tenant** (Fase 3). Sistema RAG con búsqueda híbrida, reranking, citación obligatoria y aislamiento estricto entre tenants. → *Planificado*
4. **AI Gateway multi-modelo en AWS** (Fase 5). Servicio en ECS Fargate con routing entre modelos, cache semántico, fallback automático y observabilidad de coste con Prometheus y Grafana. → *Planificado*

## Cómo navegar el repositorio

- **`bitacora/`** — Una entrada por sesión de estudio, en orden cronológico.
- **`decisions/`** — ADRs transversales que afectan a más de un proyecto (formato Michael Nygard). Aparecerá cuando exista el primero.
- **`CLAUDE.md`** — Instrucciones de trabajo para Claude Code en este repositorio.

## Contexto

- **Ubicación:** Cantabria (España). Trabajo remoto.
- **Perfil profesional:** [LinkedIn](https://www.linkedin.com/in/manueltoledano/).
- **Otros repositorios de referencia:** [task-manager-api](https://github.com/Toleflaco/task-manager-api) (monolito Spring Boot 4 con JWT, JPA, MongoDB, Testcontainers), [task-manager-microservices](https://github.com/Toleflaco/task-manager-microservices) (descomposición en microservicios con API Gateway, Resilience4j y Kafka).

---

*Última actualización: 2026-07-23*
