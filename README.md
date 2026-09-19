# AI Engineer Roadmap · Java + Spring AI

Roadmap público de mi transición de Java backend a AI Engineer, con cuatro proyectos entregables y bitácora en abierto.

## Índice de proyectos

Este es un hub. No contiene código: linkea a los cuatro repositorios de proyecto, aloja la bitácora de progreso y las decisiones arquitectónicas transversales.

**Empieza por aquí:**

1. **[document-analyzer-ai](https://github.com/Toleflaco/document-analyzer-ai)** — Spring AI + Anthropic Claude, análisis multimodal de CVs, memoria conversacional persistida en Redis. Observabilidad end-to-end (latencia, tokens, coste). **[Completado]**
    - Stack: Spring Boot 4.1, Spring AI 2.0, Claude Sonnet 4.5, Redis
    - Endpoints: `/chat` (memoria persistida), `/analyze` (texto), `/analyze/pdf` (nativo)
    - Aprendizajes: ChatClient, MessageChatMemoryAdvisor, CallAdvisor, RFC 7807

2. **[erp-mcp-server](https://github.com/Toleflaco/erp-mcp-server)** — MCP server con Spring AI que expone tooling ERP (suppliers, products, purchase orders, invoices) para agentes. Streaming HTTP transport, Flyway migrations, Docker multi-stage, CI/CD con smoke tests. **[En desarrollo]**
    - Stack: Spring Boot 4.1, Spring AI 2.0, MCP protocol (STREAMABLE), PostgreSQL, Flyway
    - Tooling: getSupplier, getProduct, createPurchaseOrder, receivePurchaseOrder, etc.
    - Aprendizajes: MCP servers, Flyway, health-gated startup, orchestration

3. **[erp-purchasing-agent](https://github.com/Toleflaco/erp-purchasing-agent)** — Agente ReAct consumidor de erp-mcp-server tools. Toma decisiones sobre reordenes automáticas basadas en stock levels. **[Pendiente]**
    - Stack: Spring Boot, Spring AI agents, Claude
    - Agentes: ReAct loop, tool calling, agentic reasoning

4. **Knowledge base empresarial (Fase 3)** — Sistema RAG con búsqueda híbrida, reranking, citación obligatoria y aislamiento estricto entre tenants. **[Planificado]**

---

## Sobre este repositorio

Soy Manuel Toledano, desarrollador Java backend. Después de reactivar mi carrera en desarrollo en 2026 con dos proyectos backend serios (monolito y microservicios sobre Spring Boot), he decidido apostar por la IA aplicada como siguiente paso lógico: no abandono Java, lo llevo al terreno donde el mercado está demandando perfiles. Este repositorio documenta ese camino en abierto.

Es un hub. No contiene código: enlaza a los cuatro repositorios de proyecto y aloja la bitácora de progreso y las decisiones arquitectónicas transversales.

## El roadmap en una tabla

| Fase | Contenido | Estado |
| --- | --- | --- |
| 0   | Prerrequisitos Java + Spring Boot sólido | ✅ Completada |
| 1   | Fundamentos LLM + Spring AI core | ✅ Completada |
| 2   | Tool calling, agentes y MCP | 🔄 En curso |
| 3   | RAG empresarial con vector stores | ⏳ Planificada |
| 4   | Observabilidad, testing y seguridad enterprise | ⏳ Planificada |
| 5   | Cloud, despliegue y LLMOps sobre AWS | ⏳ Planificada |
| 6   | Criterio arquitectónico y portfolio | ⏳ Planificada |

Cada fase termina con un proyecto entregable en un repositorio independiente. La bitácora de sesiones registra el progreso día a día. La Fase 1 se cerró técnicamente en la Sesión 08.

---

## Aprendizajes por fase

### Fase 1: Fundamentos LLM + Spring AI

**Objetivo:** dominar ChatClient, structured outputs, memoria conversacional y observabilidad básica.

**Proyecto:** [document-analyzer-ai](https://github.com/Toleflaco/document-analyzer-ai)

**Aprendizajes clave:**
- ChatClient: prompts, advisors, call flow
- BeanOutputConverter: conversión de LLM output a POJOs tipados
- MessageChatMemoryAdvisor: memoria conversacional con ventana configurable
- CallAdvisor: interceptación de llamadas LLM para observabilidad
- RFC 7807 ProblemDetail: manejo de errores estándar
- Redis: persistencia de conversaciones, TTL, serialización custom
- Testcontainers: integration tests con Redis real

**Divergencias del plan:**
- Original: memoria en Redis (✅ hecho), análisis de contratos PDF (❌ cambié a CVs), tres proveedores LLM (❌ solo Anthropic)
- Resultado: objetivos de aprendizaje cubiertos, "deuda pedagógica explícita" documentada para post-empleo

### Fase 2: Tool Calling, Agentes y MCP

**Objetivo:** dominar MCP servers, tool calling y ReAct loops.

**Proyectos:**
- [erp-mcp-server](https://github.com/Toleflaco/erp-mcp-server) — MCP server con tooling ERP
- [erp-purchasing-agent](https://github.com/Toleflaco/erp-purchasing-agent) — Agente ReAct consumidor

**Aprendizajes esperados:**
- MCP protocol: definition, streaming HTTP transport, tool discovery
- Tool calling: schema, invocation, Claude tool_use blocks
- Agents: ReAct loop, reasoning, action selection
- Spring AI agents: integration con Claude, error handling

---

## Cómo navegar el repositorio

- **`bitacora/`** — Una entrada por sesión de estudio, en orden cronológico. Actualizada hasta la [Sesión 15](bitacora/Sesion15-2026-08-05.md).
- **`decisions/`** — ADRs transversales que afectan a más de un proyecto (formato Michael Nygard).
- **`CLAUDE.md`** — Instrucciones de trabajo para Claude Code en este repositorio.

## Contexto personal

- **Ubicación:** Cantabria (España). Trabajo remoto.
- **Perfil profesional:** [LinkedIn](https://www.linkedin.com/in/manueltoledano/)
- **Otros repositorios de referencia:**
    - [task-manager-api](https://github.com/Toleflaco/task-manager-api) — Monolito Spring Boot 4 con JWT, JPA, MongoDB, Testcontainers
    - [task-manager-microservices](https://github.com/Toleflaco/task-manager-microservices) — Descomposición en microservicios con API Gateway, Resilience4j y Kafka
    - [cloud-roadmap](https://github.com/Toleflaco/cloud-roadmap) — AWS, Kubernetes, OAuth2, Observabilidad

---

*Última actualización: 2026-09-20*
