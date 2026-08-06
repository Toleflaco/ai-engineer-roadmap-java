# CLAUDE.md

Instrucciones de trabajo para Claude Code en el repositorio `ai-engineer-roadmap-java`.

---

## Propósito

Este repositorio es el índice y la bitácora de mi formación autodidacta como AI Engineer sobre stack Java + Spring AI, seguida durante 2026. No contiene código: es un hub que enlaza a cuatro repositorios independientes, uno por cada proyecto del roadmap (análisis de documentos, servidor MCP con agente, knowledge base RAG multi-tenant, y AI gateway en AWS). Aquí viven la narrativa del roadmap completo, la bitácora de progreso y las decisiones arquitectónicas transversales.

El repositorio existe con dos audiencias en mente: los reclutadores y colegas técnicos que quieran entender qué he construido y cómo lo he pensado, y yo mismo dentro de unos años, mirando atrás con honestidad al camino recorrido.

## Cómo trabajar en este repositorio

Las tareas habituales aquí son cuatro. Primera, redactar las entradas de bitácora al final de cada sesión de estudio, con el formato definido más abajo. Segunda, actualizar `README.md` cuando termine una fase del roadmap o tome una decisión que cambie el plan. Tercera, escribir ADRs en `decisions/` cuando adopte una decisión arquitectónica transversal que afecte a varios proyectos del roadmap (formato Michael Nygard, mismo que en `task-manager-api`). Cuarta, enlazar este hub con los repositorios de proyecto conforme se vayan creando, uno por fase.

**Lo que NO se hace aquí.** No se escribe código Java, ni tests, ni configuración de Spring, ni Dockerfiles. Este repositorio es documentación. Cualquier código funcional vive en los repositorios de proyecto correspondientes. Si al trabajar aquí surge la necesidad de generar código, es señal de que la tarea pertenece a otro repositorio y hay que migrar al que toque.

## Estructura del repositorio

```
ai-engineer-roadmap-java/
├── README.md              Portada pública: qué es el roadmap y estado actual
├── CLAUDE.md              Este fichero: instrucciones de trabajo
├── LICENSE                MIT
├── .gitignore
├── bitacora/              Diario de progreso, un fichero por sesión
└── decisions/             ADRs transversales (a partir de Fase 1)
```

Las carpetas `bitacora/` y `decisions/` se crean cuando existan sus primeros contenidos, no antes. Los ficheros de bitácora siguen el patrón `SesionNN-AAAA-MM-DD.md`, numerados de forma incremental global (no por fase). Las sesiones que son continuación directa de otra el mismo bloque de trabajo añaden un sufijo decimal (`SesionNN-2-AAAA-MM-DD.md`, `SesionNN-3-...`); las sesiones cortas intercaladas entre dos sesiones numeradas usan un decimal propio (`SesionNN.5-AAAA-MM-DD.md`). Los ADRs siguen el patrón `NNNN-titulo-en-kebab-case.md`, numerados de forma incremental.

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

Este hub referencia cuatro repositorios de proyecto, uno por cada proyecto del roadmap. Los nombres definitivos se decidirán al crear cada repositorio, no antes. El `README.md` de este hub mantiene enlaces actualizados a los cuatro repositorios y a su estado (planificado, en desarrollo, completado, en pausa).

Los repositorios de proyecto no dependen técnicamente de este hub: son autónomos, tienen su propio `CLAUDE.md`, sus propias convenciones (adaptadas al hecho de que son repos de código Java) y su propio ciclo de vida. La relación es solo narrativa: el hub cuenta la historia global, los repos de proyecto viven cada uno su historia local.

## Comandos y flujos habituales

Al ser un repositorio de documentación sin código, no hay comandos de build ni de test. Los flujos habituales son de Git puro:

- Añadir entrada de bitácora al final de cada sesión, en `bitacora/SesionNN-AAAA-MM-DD.md`.
- Actualizar el estado del roadmap en `README.md` cuando se cierre una fase.
- Escribir un ADR nuevo en `decisions/` cuando se adopte una decisión transversal.
- Commits directos a `main`: al ser un repo unipersonal de documentación, no hay ramas de feature ni pull requests internos.
