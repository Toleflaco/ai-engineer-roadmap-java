# AI Engineer Roadmap · Java + Spring AI

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Status](https://img.shields.io/badge/status-in%20progress-orange.svg)]()
[![Phase](https://img.shields.io/badge/phase-2%20Tool%20Calling%20%26%20MCP-blue.svg)]()
[![Java 21](https://img.shields.io/badge/Java-21-orange.svg)](https://openjdk.org/projects/jdk/21/)
[![Spring AI 2.0](https://img.shields.io/badge/Spring%20AI-2.0-blue.svg)](https://spring.io/projects/spring-ai)

Roadmap público de mi transición de Java backend a AI Engineer, estructurado en fases y con proyectos entregables por fase.

---

## Sobre este repositorio

Soy Manuel Toledano, desarrollador Java backend. Después de reactivar mi carrera en desarrollo en 2025 con dos proyectos backend serios (monolito y microservicios sobre Spring Boot), he decidido apostar por la IA aplicada como siguiente paso lógico: no abandono Java, lo llevo al terreno donde el mercado está demandando perfiles. Este repositorio documenta ese camino en abierto.

Es un hub. No contiene código: enlaza a los repositorios de proyecto y aloja las decisiones arquitectónicas transversales.

## Modelo del roadmap

El roadmap utiliza tres niveles distintos:

- **Fase:** bloque de aprendizaje con objetivos técnicos concretos.
- **Proyecto:** entregable conceptual que demuestra los objetivos de una fase.
- **Repositorio:** unidad independiente de código que implementa todo o parte de un proyecto.

Este repositorio es el **hub documental** del roadmap. No es un monorepo, no contiene código funcional y no es una dependencia técnica de los repositorios de proyecto. La relación entre ellos es narrativa y de portfolio: cada repositorio de proyecto mantiene su propio ciclo de vida, build, tests y despliegue.

## Fases de aprendizaje

| Fase | Contenido                                      | Estado         |
| ---- | ---------------------------------------------- | -------------- |
| 0    | Prerrequisitos Java + Spring Boot sólido       | ✅ Completada  |
| 1    | Fundamentos LLM + Spring AI core               | ✅ Completada  |
| 2    | Tool calling, agentes y MCP                    | 🔄 En progreso |
| 3    | RAG empresarial con vector stores              | ⏳ Planificada |
| 4    | Observabilidad, testing y seguridad enterprise | ⏳ Planificada |
| 5    | Cloud, despliegue y LLMOps sobre AWS           | ⏳ Planificada |
| 6    | Criterio arquitectónico y portfolio            | ⏳ Planificada |

Cada fase termina con uno o varios proyectos entregables en repositorios independientes.

### Fase 4: Observabilidad, testing y seguridad enterprise

La Fase 4 es transversal a los proyectos de las Fases 1, 2 y 3. No introduce un proyecto ni un repositorio independiente: su entregable consiste en consolidar las capacidades operativas y de seguridad de los proyectos existentes antes de abordar el despliegue cloud de la Fase 5.

El alcance mínimo de esta fase es:

- trazabilidad de llamadas LLM, métricas de latencia, tokens y coste;
- estrategia de testing para componentes deterministas, integraciones y flujos con modelos;
- gestión de secretos y configuración sensible;
- manejo consistente de errores y límites operativos;
- revisión de seguridad y documentación de riesgos conocidos.

### Criterios de cierre

Una fase se considera **completada** cuando sus objetivos mínimos están cubiertos, existe al menos un entregable verificable y la deuda aplazada está documentada. “Completada” no significa que se haya implementado todo el plan original sin divergencias.

| Fase | Objetivos mínimos | Entregable o evidencia | Deuda conocida |
| ---- | ----------------- | ---------------------- | -------------- |
| 0 | Demostrar una base sólida de Java y Spring Boot para abordar los proyectos del roadmap. | Proyectos backend previos documentados y utilizables como base. | Ninguna pendiente en el alcance actual. |
| 1 | Usar `ChatClient`, output estructurado, memoria conversacional, multimodalidad, manejo homogéneo de errores y observabilidad básica. | `document-analyzer-ai` funcional y documentado. | Probar proveedores LLM adicionales y recuperar el dominio original de análisis de contratos si sigue siendo relevante. |
| 2 | Implementar tool calling, exponer tools mediante MCP y construir un agente ReAct que las consuma a través de un cliente MCP. | `erp-mcp-server` operativo y `erp-purchasing-agent` capaz de completar objetivos de compras sobre el dominio ERP. | Completar el agente y documentar sus límites operativos. |
| 3 | Implementar ingestion, búsqueda híbrida, reranking, citación obligatoria y aislamiento estricto entre tenants. | Repositorio RAG independiente con documentación, pruebas de aislamiento y consultas reproducibles. | Definir el proveedor de vector store y los límites de escala cuando comience la fase. |
| 4 | Consolidar observabilidad, testing y seguridad sobre los proyectos existentes, con métricas, trazabilidad, gestión de secretos y revisión de errores. | Cambios documentados y verificables en los repositorios de Fases 1, 2 y 3; no requiere un proyecto independiente. | El alcance exacto se concretará al cerrar las fases anteriores. |
| 5 | Desplegar un servicio multi-modelo en AWS con routing, fallback, cache semántico y observabilidad de coste. | AI Gateway desplegable en ECS Fargate, con documentación operativa y evidencia de ejecución. | Definir los modelos soportados, el presupuesto de coste y la estrategia de operación. |
| 6 | Comparar alternativas, justificar decisiones transversales y presentar el portfolio con sus límites y deuda técnica. | Portfolio navegable, ADRs relevantes y revisión final de los proyectos y sus evidencias. | Ninguna hasta evaluar el conjunto completo. |

## Proyectos entregables

Los proyectos son las unidades conceptuales que demuestran los objetivos de cada fase. Un proyecto puede estar implementado por uno o varios repositorios independientes.

### Proyecto 1: Analizador de CVs con Spring AI y Anthropic Claude

**Fase:** 1 · **Estado:** Completado

Servicio Spring Boot con tres endpoints: chat con memoria conversacional persistida en Redis segmentada por `conversationId`, análisis estructurado de CV en texto plano y análisis multimodal de CV en PDF nativo. Incluye observabilidad de latencia, tokens y coste estimado por llamada mediante un advisor custom de Spring AI, además de manejo homogéneo de errores con ProblemDetail RFC 7807.

**Repositorio:**

- **[document-analyzer-ai](https://github.com/Toleflaco/document-analyzer-ai)** — Implementación completa del proyecto.

### Proyecto 2: Plataforma ERP con servidor MCP y agente ReAct

**Fase:** 2 · **Estado:** En progreso

Este proyecto está compuesto por dos repositorios independientes que forman un par: el servidor MCP expone operaciones de dominio ERP (suppliers, products, purchase orders, invoices) como tools sobre Streamable HTTP, y el agente ReAct consume esas tools mediante un cliente MCP dinámico para cumplir objetivos de compras expresados en lenguaje natural.

**Repositorios:**

- **[erp-mcp-server](https://github.com/Toleflaco/erp-mcp-server)** — Servidor MCP. Completado, con CI/CD de tres jobs (build+test, docker-build y smoke test contra el endpoint MCP).
- **[erp-purchasing-agent](https://github.com/Toleflaco/erp-purchasing-agent)** — Agente ReAct. En desarrollo.

### Proyecto 3: Knowledge base empresarial multi-tenant

**Fase:** 3 · **Estado:** Planificado

Sistema RAG con búsqueda híbrida, reranking, citación obligatoria y aislamiento estricto entre tenants.

**Repositorio:**

- Pendiente de crear.

### Proyecto 4: AI Gateway multi-modelo en AWS

**Fase:** 5 · **Estado:** Planificado

Servicio en ECS Fargate con routing entre modelos, cache semántico, fallback automático y observabilidad de coste con Prometheus y Grafana.

**Repositorio:**

- Pendiente de crear.

La Fase 4 es transversal a los proyectos anteriores y todavía no tiene un proyecto entregable independiente definido.

## Divergencias del plan original

El roadmap fusiona el Master AI Engineer de codeja.dev con un Roadmap Maestro personal. El plan de Fase 1 preveía análisis de contratos PDF, memoria conversacional persistente en Redis, y tres proveedores LLM intercambiables. Por prioridad de búsqueda activa de empleo, la Fase 1 se ha cerrado con divergencias: dominio CV en lugar de contratos, memoria in-memory sustituida más tarde por Redis (implementada en iteración posterior), un solo proveedor (Anthropic Claude Sonnet 4.5) en lugar de tres. Los objetivos de aprendizaje de la fase (ChatClient, output estructurado, memoria conversacional, multimodal, manejo de errores, observabilidad básica) quedan cubiertos.

En Fase 2, el proyecto vehículo se ha estructurado como dos repositorios en lugar de uno: `erp-mcp-server` como servidor y `erp-purchasing-agent` como cliente ReAct. La separación permite que cada pieza tenga ciclo de vida y despliegue independientes, más cercano a cómo se estructuran estos sistemas en producción real.

Estas divergencias se tratan como deuda pedagógica explícita, no como piezas terminadas. Plan post-empleo: completar los huecos originales de cada fase cuando desaparezca la presión de candidaturas.

## Cómo navegar el repositorio

- **[`projects.md`](projects.md)** — Catálogo normalizado de proyectos, repositorios, referencias y limitaciones.
- **[`CONTRIBUTING.md`](CONTRIBUTING.md)** — Guía para actualizar el estado y mantener la documentación.
- **`decisions/`** — ADRs transversales que afectan a más de un proyecto (formato Michael Nygard).
- **`CLAUDE.md`** — Instrucciones de trabajo para Claude Code en este repositorio.

## Contexto

- **Ubicación:** Cantabria (España). Trabajo remoto.
- **Perfil profesional:** [LinkedIn](https://www.linkedin.com/in/manueltoledano/).
- **Repositorios paralelos:**
    - **[cloud-roadmap](https://github.com/Toleflaco/cloud-roadmap)** — Track paralelo de cloud infrastructure (AWS, Terraform, Kubernetes, OAuth2, Observability). Complementa este roadmap con la capa de plataforma.
    - **[task-manager-api](https://github.com/Toleflaco/task-manager-api)** — Monolito Spring Boot 4 con JWT, JPA, MongoDB, Testcontainers. Proyecto vehículo para el módulo AWS del cloud-roadmap.
    - **[task-manager-microservices](https://github.com/Toleflaco/task-manager-microservices)** — Descomposición en microservicios con API Gateway, Resilience4j y Kafka. Proyecto vehículo para el módulo Kubernetes del cloud-roadmap.

---

*Última actualización: 2026-09-22*
