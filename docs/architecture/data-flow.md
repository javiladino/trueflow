# Flujo de datos — trueflow

Recorrido del dato desde la fuente hasta el consumo. Fuente: `CLAUDE.md` §2, §3, §5, §9; ADR [0001](../adr/0001-lakehouse-on-free-plan.md).
Estado: **propuesta aprobada**. Los nombres de modelos intermedios son orientativos hasta las propuestas de M1–M3.

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

    subgraph rawschema["Glue · raw (tablas externas)"]
        RE["raw.events"]
        RD["raw.datatourisme_*"]
        RV["raw.dvf_*"]
    end

    subgraph stg["staging · Iceberg (1:1, cast, dedupe)"]
        SE["stg_tracking__events<br/>dedupe event_id"]
        SD["stg_datatourisme__listings"]
        SV["stg_dvf__mutations"]
    end

    subgraph int["intermediate · Iceberg"]
        IS["int_sessions"]
        IF["int_sponsorship_steps"]
        IP["int_price_per_m2"]
    end

    subgraph marts["marts · Iceberg"]
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
        MBS["Metabase (Athena)"]
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

## 2. Secuencia de un evento

```mermaid
sequenceDiagram
    autonumber
    participant C as Simulador
    participant G as API Gateway
    participant L as Collector (Go)
    participant S as S3 raw
    participant Q as S3 quarantine / DLQ
    participant FW as Forwarder
    participant A as Amplitude
    participant AF as Airflow
    participant AT as Athena + Glue

    C->>G: POST /events (lote JSON)
    G->>L: invoke
    L->>L: validar JSON Schema, rechazar campos desconocidos
    alt inválido
        L->>Q: guardar evento + motivo
        L-->>C: 400
    else válido
        L->>L: enriquecer (received_at) + HMAC-SHA256 user_id/email
        L->>S: PutObject events/dt=/hour=/<uuid>.json
        L-->>C: 202
        S->>FW: s3:ObjectCreated
        FW->>A: envío por lotes
    end
    AF->>AT: registrar partición dt/hour (idempotente)
    AF->>AT: dbt build (dedupe event_id, lookback received_at)
    AF->>AT: OPTIMIZE / compactación Iceberg
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
    participant AT as Athena + Glue

    EB->>BI: cron diario
    BI->>DT: descarga (API key de Secrets Manager)
    BI->>BI: excluir contactos profesionales (RGPD)
    BI->>S: escribir partición dt=YYYY-MM-DD
    AF->>AT: registrar partición en raw
    AF->>AT: dbt snapshot (SCD2 dim_listing) + dbt build + tests
    AF->>AT: checks de freshness (< 26 h)
```

## 4. Controles RGPD en el flujo

| Punto | Control |
|---|---|
| Collector | HMAC-SHA256 de `user_id`/email/nombre; rechazo de campos fuera del tracking plan |
| Batch ingest | Exclusión de teléfonos/emails de profesionales; `host_name`/`reviewer_name` eliminados |
| S3 | Lifecycle: raw 13 meses, quarantine 30 días |
| Iceberg | DAG de supresión: `DELETE` vía Athena sobre `gdpr_erasure_requests` + expiración de snapshots; reescritura de objetos raw; auditado |
| Consumo | Metabase solo lectura sobre la base `marts`; texto a LLM sin PII |
