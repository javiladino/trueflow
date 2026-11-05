# trueflow — Agent Context

> **Purpose:** Orient any agent working on this project. Read this first, then stop as soon as the task is clear.
> **Last updated:** 2026-11-05

## Skills

Load by task, not all at once.

| Skill | When to load |
|-------|--------------|
| reppit-research | Explorar lo que existe antes de una feature (fase Re) |
| reppit-propose | Proponer opciones y registrar la decisión del usuario |
| reppit-plan | Plan de tareas pequeñas y ordenadas |
| reppit-implement | Ejecutar un plan aprobado por fases |
| reppit-test | Verificar criterios de aceptación con evidencia |
| grill-me | Aclarar requisitos entrevistando al usuario |
| adversarial-review | Revisar código o planes |
| design-memory | Leer, escribir o cerrar documentos en `docs/design/` |

## Active Context

**Load order:** this file → `<topic>/context.md` of the active topic → the files its document map marks `always` → only the files the task names. Never load `CHANGELOG.md`, `evidence/`, `archive/`, or a whole folder unless the task names the file.
**Active topics:** ninguno todavía.

**Current state (2026-11-05):** Repo git inicializado en `main` sin commits; `.gitignore` y symlink `.claude/skills → ../skills` creados; `docs/design/` arrancado. Solo existe la propuesta (`CLAUDE.md`) y las skills; ningún código de §4. Mapeo inicial en `docs/RESEARCH.md` (fuera de esta raíz).

**Next up:** primer commit en rama `feat/000-bootstrap`; abrir el primer topic (M0).
**Known issues:** Go, Terraform y AWS CLI no instalados en local; Python del sistema 3.9.6 (se pide 3.12).

## Conventions

- **Raíz de diseño (in force 2026-11-05; owner Javier; source bootstrap):** manda `docs/design/<topic>/working/`; la ruta `specs/<NNN>/` de CLAUDE.md §6 no se usa.
