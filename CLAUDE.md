# CLAUDE.md

Instrucciones de trabajo para Claude Code en el repositorio `ai-engineer-roadmap-java`.

---

## Propósito

Este repositorio es el índice de mi formación autodidacta como AI Engineer sobre stack Java + Spring AI, seguida durante 2026. No contiene código: es un hub que enlaza a los repositorios independientes de los proyectos del roadmap (análisis de documentos, plataforma ERP con servidor MCP y agente, knowledge base RAG multi-tenant, y AI gateway en AWS). Aquí viven la narrativa del roadmap completo y las decisiones arquitectónicas transversales. Las notas de las sesiones de estudio se mantienen fuera de este repositorio.

El repositorio existe con dos audiencias en mente: los reclutadores y colegas técnicos que quieran entender qué he construido y cómo lo he pensado, y yo mismo dentro de unos años, mirando atrás con honestidad al camino recorrido.

## Modelo del roadmap

El roadmap distingue tres niveles:

- **Fase:** bloque de aprendizaje con objetivos técnicos concretos.
- **Proyecto:** entregable conceptual que demuestra los objetivos de una fase.
- **Repositorio:** unidad independiente de código que implementa todo o parte de un proyecto.

Este repositorio es un hub documental. No es un monorepo, no contiene código funcional y no es una dependencia técnica de los repositorios de proyecto. La relación entre ellos es narrativa y de portfolio; cada repositorio de proyecto tiene su propio ciclo de vida, build, tests y despliegue.

## Cómo trabajar en este repositorio

Las tareas habituales aquí son tres: actualizar `README.md` cuando termine una fase del roadmap o tome una decisión que cambie el plan; escribir ADRs en `decisions/` cuando adopte una decisión arquitectónica transversal que afecte a varios proyectos del roadmap (formato Michael Nygard, mismo que en `task-manager-api`); y enlazar este hub con los repositorios de proyecto conforme se vayan creando o evolucionando.

**Lo que NO se hace aquí.** No se escribe código Java, ni tests, ni configuración de Spring, ni Dockerfiles. Este repositorio es documentación. Cualquier código funcional vive en los repositorios de proyecto correspondientes. Si al trabajar aquí surge la necesidad de generar código, es señal de que la tarea pertenece a otro repositorio y hay que migrar al que toque.

## Estructura del repositorio

```
ai-engineer-roadmap-java/
├── README.md              Portada pública: qué es el roadmap y estado actual
├── projects.md            Catálogo normalizado de proyectos
├── CLAUDE.md              Este fichero: instrucciones de trabajo
├── CONTRIBUTING.md        Guía de contribución y mantenimiento
├── LICENSE                MIT
├── .gitignore             Exclusiones de archivos locales
├── .github/
│   └── workflows/
│       └── documentation.yml
└── decisions/             ADRs transversales
```

El directorio `decisions/` contiene ADRs transversales. Los ADRs siguen el patrón `NNNN-titulo-en-kebab-case.md` y documentan únicamente decisiones arquitectónicas transversales que afecten a varios proyectos.

## Convenciones

Todo el contenido en español. Fechas en formato ISO 8601 (`2026-07-23`). Commits en español, sin prefijos convencionales (este repo es documentación, no tiene sentido usar `feat:` o `fix:`). Los ADRs siguen el formato Michael Nygard, con las secciones estándar: Título, Estado, Contexto, Decisión, Consecuencias.

## Instrucciones específicas para Claude Code

**Idioma.** Responder siempre en español, independientemente del idioma del prompt. Los términos técnicos consolidados en inglés (embedding, tool calling, chunking, retrieval, prompt, agente, etc.) se mantienen en inglés dentro de la frase en español.

**Tono.** Tratarme como colega técnico, no como cliente. Directo, sin ceremonias, sin adornos innecesarios. Está permitido y esperado discrepar cuando algo no cuadre, señalar errores en mi razonamiento, y llevar la contraria si hay motivos. No suavizar críticas técnicas por cortesía. Si tiene que ser duro conmigo, que lo sea. La franqueza vale más que la comodidad. Lo único que no está permitido es el desprecio: crítica dura sí, condescendencia no.

**Verbosidad.** Respuestas explicadas, con el razonamiento detrás de cada decisión. Cuando propongas una redacción, un cambio o una estructura, explica brevemente por qué. Estoy en fase de formación: el "por qué" es tan importante como el "qué". No obstante, no reciclar contexto ya establecido: si ya sabemos que este repo es documentación, no repetirlo en cada respuesta.

**Iniciativa.** Puedes tomar iniciativa proponiendo mejoras, señalando incoherencias entre ficheros del repo, o sugiriendo tareas relacionadas ("aprovechando que actualizamos el README, quizá conviene revisar X"). Pero cualquier ejecución concreta requiere mi confirmación explícita antes de tocar ficheros. La regla es: propón libremente, ejecuta solo con OK.

**Cuando dudes.** Si no tienes contexto suficiente para tomar una decisión de redacción o estructura, pregunta antes de escribir. Es preferible una pregunta a un texto que luego haya que rehacer entero. En este repo, más que en un repo de código, la precisión de matiz importa: una entrada de bitácora reescrita cinco veces pierde honestidad narrativa.

**Sobre lo que NO hacer.** No proponer código Java, tests o configuración de Spring desde este repo (para eso están los repos de proyecto). No usar prefijos de commit convencionales tipo `feat:` o `docs:`. No añadir emojis a los textos salvo que yo los use primero. No inflar la prosa con adjetivos ceremoniales ("robusto", "elegante", "moderno") cuando describa mi trabajo: hechos, no adjetivos.

## Relación con los repositorios de proyecto

Este hub referencia los repositorios de los proyectos del roadmap. Un proyecto puede estar implementado por uno o varios repositorios; por ejemplo, la plataforma ERP de la Fase 2 separa `erp-mcp-server` y `erp-purchasing-agent`. Los nombres definitivos de los repositorios futuros se decidirán al crearlos. El `README.md` y `projects.md` de este hub mantienen sus enlaces y estados actualizados (planificado, en desarrollo, completado, en pausa).

Los repositorios de proyecto no dependen técnicamente de este hub: son autónomos, tienen su propio `CLAUDE.md`, sus propias convenciones (adaptadas al hecho de que son repos de código Java) y su propio ciclo de vida. La relación es solo narrativa: el hub cuenta la historia global, los repos de proyecto viven cada uno su historia local.

## Comandos y flujos habituales

Al ser un repositorio de documentación sin código, no hay comandos de build ni de test. Los flujos habituales son de Git puro:

- Actualizar el estado del roadmap en `README.md` cuando se cierre una fase.
- Escribir un ADR nuevo en `decisions/` cuando se adopte una decisión transversal.
- Commits directos a `main`: al ser un repo unipersonal de documentación, no hay ramas de feature ni pull requests internos.
