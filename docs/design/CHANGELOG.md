# trueflow Changelog

Newest first. One `##` heading per day with activity. Write outcomes, not implementation steps. This file is appended to, never rewritten, and is not loaded by default.

## 2026-11-06

### m0-foundations — evidencia
- `evidence/` con citas de la documentación oficial y las comprobaciones de consola del owner; procedencia de findings actualizada.

### m0-foundations — plan
- Plan de M0 en 6 fases (docs, tooling, Terraform, acceso AWS, apply, CI); autenticación con IAM Identity Center; Bedrock confirmado por el usuario en el Free plan.

### m0-foundations — decisión de arquitectura
- El usuario eligió P2b (lakehouse en S3 con Athena + Glue + Iceberg, sin Kinesis/Firehose/Redshift) por Free plan, proyecto efímero y Firehose bloqueado.
- CLAUDE.md reescrito: arquitectura, stack, RGPD, roadmap y un único flujo RePPIT en `docs/design/`; nueva sección de memoria de diseño basada en `design-memory`; eliminadas las secciones duplicadas y la ruta `specs/`.
- ADR 0001, README, diagramas y `findings.md` (4 hechos) creados o actualizados.

### Files (trueflow, branch feat/001-m0-foundations)
- `CLAUDE.md`, `README.md`, `docs/architecture/*.md` (modified); `docs/adr/0001-lakehouse-on-free-plan.md`, `docs/design/findings.md` (new)

## 2026-11-05

### m0-foundations — propose
- 3 propuestas: P1 AWS completo con disciplina de coste (recomendada), P2 lakehouse Athena/Iceberg, P3 local-first. Hallazgo: Kinesis no está en el Free Tier y el Free plan cierra la cuenta a los 6 meses.

### m0-foundations — research
- Research de M0: nada de tooling/CI/Terraform/Budgets existe aún; el repo emite `sub` OIDC con IDs inmutables; S3 `use_lockfile` sustituye a DynamoDB (deprecado).

### m0-foundations — topic abierto
- Repo público en GitHub; `main` protegida; rama bootstrap borrada tras el merge.
- Research inicial movido de `docs/RESEARCH.md` a `m0-foundations/working/research.md`.

### Files (trueflow, branch feat/001-m0-foundations)
- `docs/design/m0-foundations/context.md` (new); `docs/RESEARCH.md` → `docs/design/m0-foundations/working/research.md` (moved)

### bootstrap — README y diagramas
- README del proyecto; diagramas Mermaid de arquitectura (componentes AWS) y flujo de datos (linaje por capas, secuencias evento/batch, controles RGPD), validados con mermaid-cli.

### Files (trueflow, branch feat/000-bootstrap)
- `README.md`, `docs/architecture/architecture.md`, `docs/architecture/data-flow.md` (new)

### bootstrap — arranque de repo y memoria de diseño
- Research inicial de la propuesta en `docs/RESEARCH.md`.
- Decidido: la raíz de diseño es `docs/design/` (ver CONTEXT § Conventions).
- Git inicializado en `main`, `.gitignore`, symlink `.claude/skills`, hook Stop de design-memory registrado.

### Files (trueflow, branch main, sin commit)
- `.gitignore`, `.claude/skills`, `.claude/settings.json` (new)
- `docs/design/CONTEXT.md`, `README.md`, `CHANGELOG.md` (new)
