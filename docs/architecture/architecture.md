# Arquitectura — trueflow

Lakehouse en S3 sin streaming, región **eu-west-3**, solo servicios del Free plan (ADR [0001](../adr/0001-lakehouse-on-free-plan.md)).
Fuente: `CLAUDE.md` §3. Estado: **propuesta aprobada**, nada desplegado.

```mermaid
flowchart LR
    subgraph clients["Clientes"]
        SIM["Simulador de eventos<br/>(Python)"]
    end

    subgraph ext["Fuentes externas"]
        DT["DATAtourisme API<br/>(POI 44, diario)"]
        DVF["DVF / geo-DVF<br/>(data.gouv.fr)"]
        IA_SRC["Inside Airbnb Bordeaux<br/>(opcional, fase 2)"]
    end

    subgraph aws["AWS eu-west-3 (Free plan)"]
        direction LR
        subgraph ingest["Captura"]
            APIGW["API Gateway<br/>(HTTPS)"]
            COL["Lambda collector (Go)<br/>JSON Schema · enriquecimiento · HMAC PII<br/>escritura por lotes"]
            SM[("Secrets Manager<br/>clave HMAC · API key")]
        end
        subgraph errors["Errores"]
            Q[("S3 quarantine<br/>30 días")]
            DLQ[["SQS DLQ"]]
        end
        subgraph batch["Batch"]
            EB["EventBridge Scheduler"]
            BI["Lambda batch ingest<br/>(Python)"]
        end
        RAW[("S3 raw<br/>events/dt=/hour=/<br/>13 meses")]
        FWD["Lambda forwarder<br/>(S3 ObjectCreated)"]
        subgraph lake["Lakehouse"]
            GLUE["Glue Data Catalog"]
            ICE[("S3 Iceberg<br/>staging · intermediate · marts")]
            ATH["Amazon Athena"]
        end
        BR["Bedrock (fase 2,<br/>por verificar)"]
        CW["CloudWatch"]
        BUD["AWS Budgets<br/>25 USD/mes"]
    end

    subgraph local["Local (Docker Compose)"]
        AF["Airflow 3.x<br/>particiones · dbt build · compactación · RGPD"]
        DBT["dbt-athena"]
        MB["Metabase<br/>(driver Athena, read-only marts)"]
    end

    subgraph saas["SaaS"]
        AMP["Amplitude"]
        DD["Datadog"]
    end

    SIM -->|HTTPS| APIGW --> COL
    SM -.-> COL
    COL -->|válidos, PutObject| RAW
    COL -->|inválidos| Q
    COL -->|fallos| DLQ
    RAW -->|ObjectCreated| FWD -->|lotes HTTP| AMP

    EB --> BI
    DT --> BI
    DVF --> BI
    IA_SRC -.-> BI
    SM -.-> BI
    BI --> RAW

    AF -->|registra particiones| GLUE
    AF --> DBT -->|SQL| ATH
    ATH -->|lee| RAW
    ATH -->|lee / escribe| ICE
    GLUE -.->|metadatos| ATH
    ATH --> MB
    ATH <--> BR
    CW --> DD
    AF --> DD
```

## Decisiones clave (detalle en `docs/adr/`)

| Decisión | Motivo |
|---|---|
| eu-west-3 (París) | Residencia de datos RGPD |
| Sin Kinesis, Firehose ni Redshift | No disponibles en el Free plan (ADR 0001) |
| Collector en Go, escritura directa a S3 por lotes | Cold start bajo; agrupar reduce objetos pequeños |
| Athena + Glue + Iceberg con dbt-athena | Pago por consulta; `DELETE`/`MERGE` por fila para RGPD |
| ELT con raw inmutable | Re-procesable; transformaciones versionadas en dbt |
| At-least-once + dedupe por `event_id` | Reintentos de Lambda y del cliente pueden duplicar |
| Airflow local (no MWAA) | Coste |
| SCD2 en `dim_listing` | DATAtourisme cambia a diario |
