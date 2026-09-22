# Catálogo de proyectos

Este catálogo normaliza la información de los proyectos del roadmap. La unidad
principal es el proyecto conceptual; cuando un proyecto está repartido entre
varios repositorios, cada repositorio aparece dentro de la misma ficha.

Las referencias a commits identifican el estado consultado el 2026-09-22.
Todavía no se han creado releases versionadas para estos proyectos.

## Proyecto 1: Analizador de CVs con Spring AI y Anthropic Claude

- **Fase:** 1 — Fundamentos LLM + Spring AI core
- **Estado:** Completado
- **Repositorios:** [document-analyzer-ai](https://github.com/Toleflaco/document-analyzer-ai)
- **Versión o commit de referencia:** [`675c07f398e3dfc367342e225d53f766a4e11902`](https://github.com/Toleflaco/document-analyzer-ai/commit/675c07f398e3dfc367342e225d53f766a4e11902) en `main`; release pendiente
- **Objetivo:** Aplicar `ChatClient`, output estructurado, memoria conversacional,
  multimodalidad, manejo homogéneo de errores y observabilidad básica en un
  servicio Spring Boot orientado al análisis de CVs.
- **Stack:** Java 21, Spring Boot, Spring AI 2.0, Anthropic Claude, Redis y
  ProblemDetail RFC 7807.
- **Arquitectura:** Servicio Spring Boot con endpoints independientes para chat
  conversacional, análisis estructurado de CV en texto plano y análisis
  multimodal de PDF. La memoria se segmenta por `conversationId` y la
  observabilidad se implementa mediante un advisor custom de Spring AI.
- **Evidencia:**
  [repositorio](https://github.com/Toleflaco/document-analyzer-ai)
- **Decisiones relevantes:** Priorizar el dominio CV frente al análisis de
  contratos; usar Anthropic Claude como proveedor inicial; persistir la memoria
  conversacional en Redis; instrumentar latencia, tokens y coste estimado.
- **Limitaciones conocidas:** Solo hay un proveedor LLM; el dominio original de
  contratos no forma parte del entregable actual; la release versionada está
  pendiente.

## Proyecto 2: Plataforma ERP con servidor MCP y agente ReAct

- **Fase:** 2 — Tool calling, agentes y MCP
- **Estado:** En progreso
- **Repositorios:**
  - [erp-mcp-server](https://github.com/Toleflaco/erp-mcp-server) — servidor MCP
  - [erp-purchasing-agent](https://github.com/Toleflaco/erp-purchasing-agent) —
    agente ReAct
- **Versión o commit de referencia:**
  - `erp-mcp-server`: [`94c73dc1eb68352b85b7770afb8e2a2b2a559f8c`](https://github.com/Toleflaco/erp-mcp-server/commit/94c73dc1eb68352b85b7770afb8e2a2b2a559f8c) en `main`
  - `erp-purchasing-agent`: `8448f6b1b1681680cadbfa6221f845109715db36` en `main`
  - Releases versionadas: pendientes
- **Objetivo:** Separar el proveedor de tools del consumidor agente para practicar
  tool calling, MCP y razonamiento ReAct sobre un dominio ERP.
- **Stack:** Java 21, Spring Boot, Spring AI, MCP sobre Streamable HTTP y un
  dominio ERP de suppliers, products, purchase orders e invoices.
- **Arquitectura:** Dos repositorios independientes. `erp-mcp-server` expone
  operaciones de dominio como tools MCP; `erp-purchasing-agent` se conecta
  mediante un cliente MCP dinámico y usa un agente ReAct para resolver objetivos
  de compras expresados en lenguaje natural.
- **Evidencia:**
  [servidor MCP](https://github.com/Toleflaco/erp-mcp-server) y
  [agente ReAct](https://github.com/Toleflaco/erp-purchasing-agent)
- **Decisiones relevantes:** Separar servidor y agente en repositorios con ciclos
  de vida y despliegue independientes; usar Streamable HTTP como transporte MCP;
  modelar las operaciones ERP como tools consumibles dinámicamente.
- **Limitaciones conocidas:** El agente todavía está en desarrollo; el
  comportamiento operativo y los límites del agente deben documentarse al cerrar
  la fase; aún no hay releases versionadas.

## Proyecto 3: Knowledge base empresarial multi-tenant

- **Fase:** 3 — RAG empresarial con vector stores
- **Estado:** Planificado
- **Repositorios:** Pendientes de crear
- **Versión o commit de referencia:** No aplica
- **Objetivo:** Construir una knowledge base RAG empresarial con búsqueda híbrida,
  reranking, citación obligatoria y aislamiento estricto entre tenants.
- **Stack:** Por definir; previsto sobre Java, Spring Boot y Spring AI, con un
  vector store y componentes de búsqueda híbrida todavía no seleccionados.
- **Arquitectura:** Sistema RAG multi-tenant con ingestion, retrieval híbrido,
  reranking y generación con citas. La frontera de tenant debe mantenerse en
  ingestion, almacenamiento, recuperación y respuesta.
- **Evidencia:** Especificación resumida en la
  [sección de proyectos del README](README.md#proyectos-entregables).
- **Decisiones relevantes:** Hacer del aislamiento entre tenants un requisito
  arquitectónico explícito; combinar búsqueda léxica y vectorial; exigir citas
  en las respuestas.
- **Limitaciones conocidas:** No existe implementación; faltan proveedor de
  vector store, estrategia de chunking, modelo de embeddings y criterios de
  evaluación.

## Proyecto 4: AI Gateway multi-modelo en AWS

- **Fase:** 5 — Cloud, despliegue y LLMOps sobre AWS
- **Estado:** Planificado
- **Repositorios:** Pendientes de crear
- **Versión o commit de referencia:** No aplica
- **Objetivo:** Crear un gateway que enrute peticiones entre modelos, aplique
  fallback y cache semántico, y haga visible el coste de las llamadas LLM.
- **Stack:** AWS ECS Fargate, Prometheus y Grafana; modelos, framework de
  routing y proveedor de cache pendientes de selección.
- **Arquitectura:** Servicio gateway desplegable en ECS Fargate con routing
  multi-modelo, fallback automático, cache semántico y observabilidad operativa
  y de coste.
- **Evidencia:** Especificación resumida en la
  [sección de proyectos del README](README.md#proyectos-entregables).
- **Decisiones relevantes:** Ejecutar el gateway en ECS Fargate; tratar el
  routing, el fallback, la cache y el coste como capacidades explícitas del
  servicio.
- **Limitaciones conocidas:** No existe implementación; faltan modelos
  soportados, presupuesto de coste, política de fallback, estrategia de cache y
  diseño operativo.

## Fase transversal 4

La Fase 4 no tiene una ficha de proyecto porque no introduce un entregable
independiente. Se aplica sobre los proyectos de las Fases 1, 2 y 3 y cubre
observabilidad, testing y seguridad enterprise. Sus evidencias se registrarán
en los repositorios afectados cuando la fase se ejecute.
