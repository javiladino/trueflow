# Flujo de datos — trueflow

Recorrido del dato desde la fuente hasta el consumo. Fuente: `CLAUDE.md` §2, §3, §5, §8. Estado: **propuesta**.

## 1. Linaje por capas

```mermaid
flowchart LR
    subgraph src["Fuentes"]
        E["Eventos de producto<br/>(simulados, 9 tipos)"]
        D["DATAtourisme<br/>alojamientos/POI 44"]
        V["DVF 44<br/>transacciones"]
        R["Inside Airbnb<br/>reseñas (fase 2)"]
    end

    subgraph s3["S3 raw (inmutable)"]
        S3E["events/dt=/hour=/"]
        S3D["datatourisme/dt=/"]
        S3V["dvf/"]
    end

    subgraph rawschema["Redshift · raw"]
        RE["raw.events"]
        RD["raw.datatourisme_*"]
        RV["raw.dvf_*"]
    end

    subgraph stg["staging (1:1, cast, dedupe)"]
        SE["stg_tracking__events<br/>dedupe event_id"]
        SD["stg_datatourisme__listings"]
        SV["stg_dvf__mutations"]
    end

    subgraph int["intermediate"]
        IS["int_sessions"]
        IF["int_sponsorship_steps"]
        IP["int_price_per_m2"]
    end

    subgraph marts["marts"]
        FE["fct_events"]
        FS["fct_sessions"]
        DL["dim_listing (SCD2)"]
        DM["dim_member"]
        DC["dim_commune"]
        MF["mart_sponsorship_funnel"]
        MT["mart_listing_trust_signals"]
        MP["mart_real_estate_prices"]
        MS["mart_pipeline_slo"]
    end

    subgraph use["Consumo"]
        MBS["Metabase"]
        AI["Módulo IA"]
        DDS["Datadog"]
    end

    E --> S3E --> RE --> SE
    D --> S3D --> RD --> SD
    V --> S3V --> RV --> SV
    R -.-> AI

    SE --> IS --> FS
    SE --> FE
    SE --> IF --> MF
    SE --> DM
    SD -->|snapshot| DL
    SD --> DC
    SV --> IP --> MP
    SV --> DC
    DL --> MT
    FE --> MT
    FE --> MS

    marts --> MBS
    DL --> AI
    MS --> DDS
```

## 2. Secuencia de un evento (camino caliente)

```mermaid
sequenceDiagram
    autonumber
    participant C as Simulador
    participant G as API Gateway
    participant L as Collector (Go)
    participant K as Kinesis
    participant F as Firehose
    participant S as S3 raw
    participant Q as S3 quarantine / DLQ
    participant A as Amplitude
    participant AF as Airflow
    participant R as Redshift

    C->>G: POST /events (JSON)
    G->>L: invoke
    L->>L: validar JSON Schema, rechazar campos desconocidos
    alt inválido
        L->>Q: guardar evento + motivo
        L-->>C: 400
    else válido
        L->>L: enriquecer (received_at) + HMAC-SHA256 user_id/email
        L->>K: PutRecord
        L-->>C: 202
        K->>F: stream
        F->>S: buffer → objeto en dt=/hour=/
        K->>A: forwarder (lotes)
    end
    AF->>R: COPY con manifest (idempotente)
    AF->>R: dbt build (dedupe event_id, lookback received_at)
```

## 3. Camino batch (diario)

```mermaid
sequenceDiagram
    autonumber
    participant EB as EventBridge
    participant BI as Lambda batch ingest
    participant DT as DATAtourisme / DVF
    participant S as S3 raw
    participant AF as Airflow
    participant R as Redshift

    EB->>BI: cron diario
    BI->>DT: descarga (API key de Secrets Manager)
    BI->>BI: excluir contactos profesionales (RGPD)
    BI->>S: escribir partición dt=YYYY-MM-DD
    AF->>R: COPY → raw
    AF->>R: dbt snapshot (SCD2 dim_listing) + dbt build + tests
    AF->>R: checks de freshness (< 26 h)
```

## 4. Controles RGPD en el flujo

| Punto | Control |
|---|---|
| Collector | HMAC-SHA256 de `user_id`/email/nombre; rechazo de campos fuera del tracking plan |
| Batch ingest | Exclusión de teléfonos/emails de profesionales; `host_name`/`reviewer_name` eliminados |
| S3 | Lifecycle: raw 13 meses, quarantine 30 días |
| Redshift | DAG de supresión sobre `gdpr_erasure_requests`, auditado; reescritura de particiones S3 |
| Consumo | Metabase solo lectura sobre `marts`; texto a LLM sin PII |
