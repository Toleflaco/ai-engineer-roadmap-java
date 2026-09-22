# Contribuir al roadmap

Este repositorio es un hub documental. No contiene código funcional, builds ni
tests de Java; esos artefactos pertenecen a los repositorios de proyecto.

## Actualizar el estado de un proyecto

Cuando un proyecto cambie de estado:

1. Actualiza `README.md` y la ficha correspondiente de `projects.md`.
2. Indica la fase, el estado y el repositorio o repositorios afectados.
3. Asocia los proyectos completados a una release o a un commit de referencia
   verificable.
4. Describe la evidencia disponible y la deuda que permanezca abierta.
5. Actualiza la fecha de última modificación de `README.md`.

## Añadir decisiones

Crea un ADR en `decisions/` únicamente cuando la decisión afecte a varios
proyectos del roadmap. Usa el formato Michael Nygard y el patrón de nombre
`NNNN-titulo-en-kebab-case.md`.

Las decisiones internas de un único proyecto deben documentarse en el
repositorio de ese proyecto, no en este hub.

## Revisar cambios

Antes de confirmar un cambio:

- comprueba que los enlaces Markdown apuntan a destinos válidos;
- evita añadir código funcional, configuración de Spring o artefactos de build;
- verifica que la terminología distingue entre fase, proyecto y repositorio;
- documenta explícitamente cualquier deuda o divergencia del plan original.

El workflow de GitHub Actions valida automáticamente los enlaces Markdown en
pushes a `main` y en pull requests.
