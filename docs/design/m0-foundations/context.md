# m0-foundations — Context

**Tracker:** none
**Status:** In progress · **Owner:** Javier

## Orientation

Hito M0 (CLAUDE.md §12): arranque del repo — tooling (uv, ruff, mypy, Go, Terraform), pre-commit, esqueleto de CI
en GitHub Actions con OIDC, backend remoto de Terraform (S3 + locking) y alarma de AWS Budgets.
Restricciones: nada facturable en AWS sin confirmación explícita (CLAUDE.md §12); cuenta en Free plan; 25 USD/mes. Fase actual: Plan redactado, pendiente de aprobación.

## Rulings

- **Raíz de diseño (Javier, 2026-11-05 16:45):** Verdict: per project CONTEXT Conventions ("Raíz de diseño").
- **Arquitectura (Javier, 2026-11-06 10:00):** Verdict: per project CONTEXT Conventions ("Arquitectura lakehouse (P2b)"). Topic detail: M0 prepara bootstrap del bucket de estado, backend `use_lockfile`, rol OIDC con `sub` inmutable y budget; no crea recursos de datos.
- **Presupuesto (Javier, 2026-11-05 18:00):** budget mensual de 25 USD con alertas por email; el email va en `*.tfvars` local, no en el repo.

## Journal (newest first)

**2026-11-06 (2):** `working/plan.md` con 6 fases y 14 checks. Grill-me: autenticación local → IAM Identity Center. El usuario confirmó que Bedrock está en el Free plan (se aplica en la fase 1).

**2026-11-06:** El usuario eligió P2b. Decision registrada; ruling de arquitectura en CONTEXT; CLAUDE.md (§3–13 reescritos y memoria de diseño apuntando a `design-memory`), README, diagramas y ADR 0001 actualizados; `findings.md` creado con 4 hechos. Close-out: ver CHANGELOG.

**2026-11-05:** Topic abierto; research de M0 (§7–15) con respuestas del usuario (25 USD/mes, versiones, checks obligatorios); 3 propuestas y grill-me (seguir en Free plan → P1 descartada; Firehose bloqueado → P2b).

## Document map

| File | Role | Load | What it holds |
|------|------|------|---------------|
| `working/plan.md` | draft | always | Plan de M0: 6 fases, ficheros, 14 checks para reppit-test |
| `working/proposal.md` | draft | on-demand | Propuestas P1, P2, P2b, P3 bajo 25 USD/mes y Free plan; grill-me y Decision (P2b) |
| `working/research.md` | draft | on-demand | Mapeo general (§1–6) y estado de los cimientos de M0 con docs oficiales y respuestas del usuario (§7–15) |
