# trueflow

**Owner:** Javier Ladino
**Tracker:** none
**Status:** Propuesta; sin código todavía

## Summary

Plataforma de datos end-to-end para un marketplace de anuncios basado en la confianza (padrinazgo) en
Nantes / Loire-Atlantique: tracking de eventos, ingesta batch (DATAtourisme, DVF), warehouse en Redshift
con dbt, dashboards y módulo de IA. Proyecto portfolio para entrevista de Data Engineer.

## Phases

| Phase | Outcome | Status |
|-------|---------|--------|
| **M0** — Arranque | Tooling, pre-commit, CI, backend Terraform, alarma de presupuesto | Planned - no topic |
| **M1** — Ingesta batch | DATAtourisme + DVF → S3 → Redshift raw → staging dbt | Planned - no topic |
| **M2** — Tracking + collector | Tracking plan, collector Go, Kinesis/Firehose, DLQ | Planned - no topic |
| **M3** — Orquestación y marts | DAGs Airflow, dbt build, calidad, marts | Planned - no topic |
| **M4** — Consumo | Metabase + forwarder Amplitude | Planned - no topic |
| **M5** — Observabilidad | Datadog como código, mart de SLOs, runbooks | Planned - no topic |
| **M6** — RGPD | Seudonimización, DAG de supresión, retención | Planned - no topic |
| **M7** — IA | Enriquecimiento LLM + evaluación | Planned - no topic |

## Documents

| Topic | Document | Path |
|-------|----------|------|
| — | Agent context | [CONTEXT.md](CONTEXT.md) |
| — | Changelog | [CHANGELOG.md](CHANGELOG.md) |
| — | Arquitectura (Mermaid) | [architecture.md](../architecture/architecture.md) |
| — | Flujo de datos (Mermaid) | [data-flow.md](../architecture/data-flow.md) |
