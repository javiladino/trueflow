# trueflow

Plataforma de datos end-to-end para un **marketplace de anuncios basado en la confianza**: los miembros entran
por padrinazgo y publican anuncios de alquiler de temporada, inmobiliario y servicios en **Nantes y
Loire-Atlantique (44)**.

> Proyecto portfolio de Data Engineering. Cada decisión técnica se explica, se mide y se documenta.
> **Estado:** propuesta y arranque del repo (M0). Todavía no hay código ni infraestructura desplegada.

## Qué resuelve

Dar a Producto, Marketing y Product Ops un acceso a los datos **estable, seguro y observable**: desde el tracking
de eventos hasta los dashboards y las funcionalidades con IA.

## Arquitectura

```mermaid
flowchart LR
    SIM[Simulador] --> APIGW[API Gateway] --> COL[Collector Go]
    COL -->|válidos| KDS[Kinesis] --> FH[Firehose] --> RAW[(S3 raw)]
    COL -->|inválidos| Q[(Quarantine + DLQ)]
    KDS --> AMP[Amplitude]
    SRC[DATAtourisme · DVF] --> BI[Lambda batch] --> RAW
    RAW --> AF[Airflow + dbt] --> RS[(Redshift Serverless)]
    RS --> MB[Metabase]
    RS --> AI[Módulo IA]
    AF --> DD[Datadog]
```

- Diagrama completo: [docs/architecture/architecture.md](docs/architecture/architecture.md)
- Flujo de datos (linaje, secuencias, RGPD): [docs/architecture/data-flow.md](docs/architecture/data-flow.md)

## Fuentes de datos

| Fuente | Uso |
|---|---|
| [DATAtourisme](https://www.datatourisme.fr/) (Licence Ouverte 2.0) | Catálogo de alojamientos en el 44, ingesta diaria, historizado SCD2 |
| [DVF / geo-DVF](https://www.data.gouv.fr/) | Transacciones inmobiliarias, precio/m² por barrio |
| Inside Airbnb Bordeaux (opcional) | Texto de reseñas para el módulo de IA |
| Simulador (`simulator/`) | Eventos de producto sintéticos anclados en IDs reales |

No se hace scraping de ninguna plataforma.

## Stack

AWS (eu-west-3): API Gateway, Lambda, Kinesis + Firehose, S3, SQS, EventBridge, Redshift Serverless,
Secrets Manager, Bedrock · Terraform · Go · Python 3.12 + uv · dbt-redshift · Airflow 3.x · Metabase ·
Datadog · Amplitude · GitHub Actions (OIDC).

## Principios

- **ELT** con raw inmutable y re-procesable; transformaciones en dbt (staging → intermediate → marts).
- **At-least-once** con deduplicación por `event_id`; eventos tardíos con ventana de lookback.
- **RGPD por diseño**: seudonimización HMAC en el collector, minimización, retención y derecho de supresión auditado.
- **SLOs medidos**: disponibilidad, tasa de error, latencia p50/p95, freshness y coste en `mart_pipeline_slo` y Datadog.

## Roadmap

| Hito | Contenido |
|---|---|
| M0 | Tooling, pre-commit, CI, backend Terraform, alarma de presupuesto |
| M1 | Ingesta batch DATAtourisme + DVF → Redshift raw → staging dbt |
| M2 | Tracking plan + collector Go + Kinesis/Firehose, DLQ/quarantine |
| M3 | DAGs Airflow, calidad de datos, marts |
| M4 | Dashboards Metabase + forwarder Amplitude |
| M5 | Observabilidad: Datadog como código, runbooks |
| M6 | RGPD: DAG de supresión, retención |
| M7 | Módulo IA con evaluación offline |

## Comandos (previstos)

Aún no existe el `Makefile`; estos son los comandos planificados:

```
make setup | data-download | test | lint | up | down | tf-plan ENV=dev | simulate RATE=50 | dbt-build | destroy ENV=dev
```

## Cómo se trabaja

Flujo **RePPIT** (Research → Propose → Plan → Implement → Test) con validación humana en cada fase; los
documentos de diseño viven en [`docs/design/`](docs/design/README.md). Instrucciones para agentes en
[`CLAUDE.md`](CLAUDE.md).
