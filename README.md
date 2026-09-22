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

## El roadmap en una tabla

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

## Los proyectos

1. **Analizador de CVs con Spring AI y Anthropic Claude** (Fase 1). Servicio Spring Boot con tres endpoints: chat con memoria conversacional persistida en Redis segmentada por conversationId, análisis estructurado de CV en texto plano, y análisis multimodal de CV en PDF nativo. Observabilidad de latencia, tokens y coste estimado por llamada mediante advisor custom de Spring AI. Manejo de errores homogéneo con ProblemDetail RFC 7807. → **[document-analyzer-ai](https://github.com/Toleflaco/document-analyzer-ai)** — Completado.

2. **Servidor MCP + agente ReAct sobre dominio ERP** (Fase 2). Dos repositorios que forman un par: un servidor MCP que expone operaciones de dominio ERP (suppliers, products, purchase orders, invoices) como tools sobre Streamable HTTP, y un agente ReAct que consume esas tools vía cliente MCP dinámico para cumplir objetivos de compras expresados en lenguaje natural.
    - → **[erp-mcp-server](https://github.com/Toleflaco/erp-mcp-server)** — Servidor MCP. Completado, con CI/CD de tres jobs (build+test, docker-build, smoke test contra el endpoint MCP).
    - → **[erp-purchasing-agent](https://github.com/Toleflaco/erp-purchasing-agent)** — Agente ReAct. En desarrollo.

3. **Knowledge base empresarial multi-tenant** (Fase 3). Sistema RAG con búsqueda híbrida, reranking, citación obligatoria y aislamiento estricto entre tenants. → *Planificado.*

4. **AI Gateway multi-modelo en AWS** (Fase 5). Servicio en ECS Fargate con routing entre modelos, cache semántico, fallback automático y observabilidad de coste con Prometheus y Grafana. → *Planificado.*

## Divergencias del plan original

El roadmap fusiona el Master AI Engineer de codeja.dev con un Roadmap Maestro personal. El plan de Fase 1 preveía análisis de contratos PDF, memoria conversacional persistente en Redis, y tres proveedores LLM intercambiables. Por prioridad de búsqueda activa de empleo, la Fase 1 se ha cerrado con divergencias: dominio CV en lugar de contratos, memoria in-memory sustituida más tarde por Redis (implementada en iteración posterior), un solo proveedor (Anthropic Claude Sonnet 4.5) en lugar de tres. Los objetivos de aprendizaje de la fase (ChatClient, output estructurado, memoria conversacional, multimodal, manejo de errores, observabilidad básica) quedan cubiertos.

En Fase 2, el proyecto vehículo se ha estructurado como dos repositorios en lugar de uno: `erp-mcp-server` como servidor y `erp-purchasing-agent` como cliente ReAct. La separación permite que cada pieza tenga ciclo de vida y despliegue independientes, más cercano a cómo se estructuran estos sistemas en producción real.

Estas divergencias se tratan como deuda pedagógica explícita, no como piezas terminadas. Plan post-empleo: completar los huecos originales de cada fase cuando desaparezca la presión de candidaturas.

## Cómo navegar el repositorio

- **`decisions/`** — ADRs transversales que afectan a más de un proyecto (formato Michael Nygard). Aparecerá cuando exista el primero.
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
