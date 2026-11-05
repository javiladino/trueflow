# Arquitectura — trueflow

Vista de componentes y límites de AWS (región **eu-west-3**). Fuente: `CLAUDE.md` §3. Estado: **propuesta**, nada desplegado.

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

    subgraph aws["AWS eu-west-3"]
        direction LR
        subgraph ingest["Captura"]
            APIGW["API Gateway<br/>(HTTPS)"]
            COL["Lambda collector (Go)<br/>JSON Schema · enriquecimiento · HMAC PII"]
            SM[("Secrets Manager<br/>clave HMAC · API key")]
        end
        subgraph stream["Streaming"]
            KDS["Kinesis Data Stream"]
            FH["Kinesis Firehose"]
            FWD["Lambda forwarder"]
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
        RS[("Redshift Serverless<br/>raw → staging → intermediate → marts")]
        BR["Bedrock (fase 2)"]
        CW["CloudWatch"]
    end

    subgraph local["Local (Docker Compose)"]
        AF["Airflow 3.x<br/>COPY · dbt build · calidad · RGPD"]
        DBT["dbt-redshift"]
        MB["Metabase<br/>(rol read-only marts)"]
    end

    subgraph saas["SaaS"]
        AMP["Amplitude"]
        DD["Datadog"]
    end

    SIM -->|HTTPS| APIGW --> COL
    SM -.-> COL
    COL -->|válidos| KDS
    COL -->|inválidos| Q
    COL -->|fallos| DLQ
    KDS --> FH --> RAW
    KDS --> FWD -->|lotes HTTP| AMP

    EB --> BI
    DT --> BI
    DVF --> BI
    IA_SRC -.-> BI
    SM -.-> BI
    BI --> RAW

    AF -->|COPY con manifest| RS
    RAW -.->|lectura| RS
    AF --> DBT --> RS
    RS --> MB
    RS <--> BR
    CW --> DD
    AF --> DD
```

## Decisiones clave (resumen; detalle futuro en `docs/adr/`)

| Decisión | Motivo |
|---|---|
| eu-west-3 (París) | Residencia de datos RGPD |
| Collector en Go | Cold start bajo, validación tipada |
| ELT con raw inmutable | Re-procesable; transformaciones versionadas en dbt |
| At-least-once + dedupe por `event_id` | Kinesis/Firehose pueden duplicar |
| Airflow local (no MWAA) | Coste |
| SCD2 en `dim_listing` | DATAtourisme cambia a diario |
