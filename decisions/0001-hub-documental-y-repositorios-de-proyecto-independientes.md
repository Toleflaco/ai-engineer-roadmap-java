# ADR 0001: Mantener el roadmap como hub documental con repositorios independientes

## Estado

Aceptada

## Contexto

El roadmap contiene varios proyectos Java y Spring AI con responsabilidades,
dependencias, ciclos de vida y despliegues diferentes. El repositorio principal
necesita documentar la evolución global sin convertirse en un monorepo de código
ni introducir dependencias técnicas entre los proyectos.

Algunos proyectos pueden estar compuestos por más de un repositorio. Es el caso
de la plataforma ERP de la Fase 2, que separa el servidor MCP del agente ReAct.
Esa separación representa una frontera arquitectónica real y permite gestionar
cada pieza de forma independiente.

## Decisión

Mantener `ai-engineer-roadmap-java` como hub documental del roadmap. Cada
proyecto funcional vivirá en uno o varios repositorios independientes, según sus
fronteras y necesidades de ciclo de vida.

La relación entre este hub y los repositorios de proyecto será narrativa y de
portfolio, no una dependencia técnica de código o build. Este repositorio será
responsable de:

- documentar las fases y sus criterios de cierre;
- catalogar los proyectos y sus repositorios;
- enlazar versiones o commits de referencia;
- registrar decisiones arquitectónicas transversales.

Los builds, tests, configuración de ejecución y despliegues permanecerán en los
repositorios de proyecto correspondientes.

## Consecuencias

### Positivas

- Cada proyecto puede evolucionar, probarse y desplegarse de forma independiente.
- Las fronteras entre componentes distribuidos, como servidor MCP y agente,
  quedan representadas también en la organización del código.
- El hub mantiene un alcance pequeño y centrado en la narrativa, el estado y las
  decisiones transversales.
- Los repositorios funcionales no necesitan conocer ni importar este hub.

### Negativas

- El estado del portfolio depende de mantener actualizados los enlaces, estados y
  referencias de commits en este repositorio.
- La visión completa de un proyecto repartido entre varios repositorios exige
  consultar más de una ubicación.
- Las decisiones que afecten a varios proyectos deben sincronizarse
  documentalmente mediante ADRs.

### Deuda y límites

- Cada proyecto completado debe asociarse a una release o a un commit de
  referencia verificable.
- No se centralizarán builds ni tests en este repositorio.
- Las decisiones internas de un único proyecto deben documentarse en su propio
  repositorio, no en este ADR.
