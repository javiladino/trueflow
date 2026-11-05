# m0-foundations — Context

**Tracker:** none
**Status:** In progress · **Owner:** Javier

## Orientation

Hito M0 (CLAUDE.md §12): arranque del repo — tooling (uv, ruff, mypy, Go, Terraform), pre-commit, esqueleto de CI
en GitHub Actions con OIDC, backend remoto de Terraform (S3 + locking) y alarma de AWS Budgets.
Restricción: nada facturable en AWS sin confirmación explícita (CLAUDE.md §11). Fase actual: Research.

## Rulings

- **Raíz de diseño (Javier, 2026-11-05 16:45):** Verdict: per project CONTEXT Conventions ("Raíz de diseño").

## Journal (newest first)

**2026-11-05 (2):** Research de M0 añadido a `working/research.md` §7–14: ninguno de los cinco componentes existe; el repo usa `sub` OIDC inmutable; locking DynamoDB deprecado; Budgets gratis. Pendiente 🚦 del usuario.

**2026-11-05:** Topic abierto. El mapeo inicial de la propuesta (antes `docs/RESEARCH.md`) pasa a `working/research.md`; aún no cubre las decisiones propias de M0. `main` protegida (PR obligatorio, historia lineal).

## Document map

| File | Role | Load | What it holds |
|------|------|------|---------------|
| `working/research.md` | draft | always | Mapeo general (§1–6) y estado de los cimientos de M0 con docs oficiales (§7–14) |
