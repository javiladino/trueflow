# trueflow

**Owner:** Javier Ladino
**Tracker:** none
**Status:** M0 en curso; arquitectura aprobada (ADR 0001); sin código todavía

## Summary

Plataforma de datos end-to-end para un marketplace de anuncios basado en la confianza (padrinazgo) en
Nantes / Loire-Atlantique: tracking de eventos, ingesta batch (DATAtourisme, DVF), lakehouse en S3 (Athena + Iceberg)
con dbt, dashboards y módulo de IA. Proyecto portfolio para entrevista de Data Engineer.

## Phases

| Phase | Outcome | Status |
|-------|---------|--------|
| **M0** — Arranque | Tooling, pre-commit, CI, backend Terraform, alarma de presupuesto | In progress |
| **M1** — Ingesta batch | DATAtourisme + DVF → S3 raw → Athena/Glue → staging dbt | Planned - no topic |
| **M2** — Tracking + collector | Tracking plan, collector Go → S3 por lotes, DLQ | Planned - no topic |
| **M3** — Orquestación y marts | DAGs Airflow, dbt build, compactación Iceberg, calidad, marts | Planned - no topic |
| **M4** — Consumo | Metabase + forwarder Amplitude | Planned - no topic |
| **M5** — Observabilidad | Datadog como código, mart de SLOs, runbooks | Planned - no topic |
| **M6** — RGPD | Seudonimización, DAG de supresión, retención | Planned - no topic |
| **M7** — IA | Enriquecimiento LLM + evaluación | Planned - no topic |

## Topics

| Topic | Title | Status |
|-------|-------|--------|
| [`m0-foundations`](m0-foundations/context.md) | M0 — Arranque del repo | In progress |

## Documents

| Topic | Document | Path |
|-------|----------|------|
| — | Agent context | [CONTEXT.md](CONTEXT.md) |
| — | Changelog | [CHANGELOG.md](CHANGELOG.md) |
| — | Arquitectura (Mermaid) | [architecture.md](../architecture/architecture.md) |
| — | Flujo de datos (Mermaid) | [data-flow.md](../architecture/data-flow.md) |
| — | Findings | [findings.md](findings.md) |
| — | ADR 0001 lakehouse | [0001-lakehouse-on-free-plan.md](../adr/0001-lakehouse-on-free-plan.md) |
