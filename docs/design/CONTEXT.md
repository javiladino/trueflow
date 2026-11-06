# trueflow — Agent Context

> **Purpose:** Orient any agent working on this project. Read this first, then stop as soon as the task is clear.
> **Last updated:** 2026-11-06

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
**Active topics:** `m0-foundations` (Plan redactado; pendiente 🚦).
**Known findings:** `findings.md` — 4 entries, last 2026-11-06. Open it only when the task touches a subject it covers.

**Current state (2026-11-06):** Rama `feat/001-m0-foundations`. Arquitectura elegida: ruling "Arquitectura lakehouse (P2b)"; CLAUDE.md, README, diagramas y ADR 0001 reescritos en consecuencia. Cuenta AWS en Free plan; nada desplegado.

**Next up:** aprobación del plan → `reppit-implement` (fases 1–3 sin AWS).
**Known issues:** Go, Terraform y AWS CLI no instalados en local; sin verificar `dbt snapshot`/unit tests en `dbt-athena` y Bedrock en el Free plan.

## Conventions

- **Raíz de diseño (in force 2026-11-05; owner Javier; source bootstrap):** los documentos de diseño viven en `docs/design/<topic>/` (fases RePPIT en `working/`); no se usa `specs/`.
- **Arquitectura lakehouse (P2b) (in force 2026-11-06; owner Javier; source m0-foundations):** collector → S3 directo por lotes; Athena + Glue + Iceberg con `dbt-athena`; sin Kinesis, Firehose ni Redshift; solo servicios del Free plan, proyecto efímero. Detalle: `docs/adr/0001-lakehouse-on-free-plan.md`; discusión: `m0-foundations/working/proposal.md` § Decision.
- **m0-foundations-scoped rulings** live in `m0-foundations/context.md` § Rulings.
