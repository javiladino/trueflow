# CLAUDE.md — Plataforma de datos para un marketplace de confianza (proyecto portfolio)

## 1. Propósito

Plataforma de datos end-to-end para un **marketplace de anuncios basado en la confianza** (los miembros
entran por padrinazgo / "parrainage"; verticales principales: alquiler de temporada, inmobiliario, servicios).
Ámbito geográfico: **Nantes y Loire-Atlantique (44)**.

Es un proyecto de preparación para una entrevista de Data Engineer: cada decisión técnica debe ser
**explicable, medida y documentada**, no solo funcionar.

Objetivo principal: dar a Producto, Marketing y Product Ops un **acceso a los datos estable, seguro y
observable**, desde el tracking de eventos hasta los dashboards y las funcionalidades con IA.

Idioma: documentación y conversación en **español** (por ahora). Identificadores de código, nombres de
tablas/columnas, commits y mensajes de log en **inglés**.

## 2. Fuentes de datos (todas reales, salvo el simulador de eventos)

| Fuente | Contenido | Acceso | Uso |
|---|---|---|---|
| **DATAtourisme** (ADN Tourisme) | Puntos de interés turísticos geolocalizados: **alojamientos**, restaurantes, actividades, eventos. Actualización diaria. Licence Ouverte 2.0 | API con clave (datatourisme.fr) o exports de data.gouv.fr | Catálogo de "anuncios" de alquiler de temporada en Nantes/44; ingesta incremental diaria |
| **DVF / geo-DVF** (data.gouv.fr) | Transacciones inmobiliarias reales, fichero por departamento (44) | Descarga de ficheros | Vertical inmobiliario, precio/m² por barrio, detección de precios anómalos |
| **Inside Airbnb — Bordeaux** (opcional) | Anuncios, calendario y reseñas reales (texto FR/EN) | Descarga CSV | Solo como fuente de **texto de reseñas** para el módulo de IA (fase 2). No hay dataset de Nantes |
| **Simulador de eventos** (`simulator/`) | Eventos de producto sintéticos **anclados en IDs reales** de DATAtourisme y DVF | Generado | Pipeline de tracking (no existen datos públicos de eventos de producto) |

**Prohibido**: hacer scraping de Airbnb o de cualquier otra plataforma (viola sus condiciones de uso).
Airbnb no tiene API pública.

Los eventos simulados siguen el tracking plan definido en `tracking-plan/` (JSON Schema). Eventos principales:
`signup_started`, `sponsorship_requested`, `sponsorship_accepted`, `member_validated`,
`listing_viewed`, `listing_published`, `contact_requested`, `booking_requested`, `review_posted`.

Los ficheros raw **nunca se commitean**. Se descargan con `make data-download` (scripts en `ingestion/batch/`).
Las claves de API (DATAtourisme) van en Secrets Manager / `.env` local, nunca en el repo.

## 3. Arquitectura

```
[Simulador / clientes] --HTTPS--> API Gateway --> Lambda collector (Go)
        - validación JSON Schema (tracking plan)  - enriquecimiento  - seudonimización de PII
              |  válidos                                  | inválidos
              v                                           v
        Kinesis Data Stream                        S3 quarantine + SQS DLQ
              |                     \
              v                      \--> Lambda forwarder --> Amplitude HTTP API (por lotes)
        Kinesis Firehose --> S3 raw  (s3://<bucket>/events/dt=YYYY-MM-DD/hour=HH/)
                                  |
EventBridge Scheduler --> Lambda batch ingest (DATAtourisme diario, DVF) --> S3 raw
                                  |
                       Airflow (orquestación)
                        - COPY S3 -> Redshift esquema `raw` (idempotente, con manifest)
                        - dbt build (staging -> intermediate -> marts) + tests
                        - checks de calidad / freshness, job de borrado RGPD
                                  |
                       Redshift Serverless (eu-west-3)
                                  |
              Metabase (dashboards)   Datadog (monitores del pipeline)   Módulo IA (tablas de evaluación)
```

Decisiones clave (cualquier cambio se registra como ADR en `docs/adr/`):
- Región **eu-west-3 (París)**: argumento de residencia de datos para el RGPD.
- Collector en **Go** (cold start bajo, validación tipada); batch y orquestación en **Python**.
- **ELT**: se aterriza en raw y se transforma en el warehouse con dbt. Raw es inmutable y re-procesable.
- Entrega at-least-once → **deduplicación por `event_id`** en staging de dbt.
- Eventos tardíos: modelos incrementales con ventana de lookback sobre `received_at`.
- DATAtourisme cambia a diario → snapshot dbt (**SCD2**) de `dim_listing` para historizar cambios.

## 4. Estructura del repositorio

```
.
├── CLAUDE.md
├── .claude/skills/            # skills RePPIT + skills de dominio (ver §6)
├── specs/<NNN-feature>/       # research.md, spec.md, plan.md, impl-log.md, verify.md
├── docs/adr/                  # Architecture Decision Records (NNNN-titulo.md)
├── docs/interview/            # preguntas/respuestas y trade-offs por feature
├── docs/runbooks/             # qué hacer cuando salta cada alerta
├── tracking-plan/             # JSON Schemas de eventos + changelog
├── collector/                 # Lambda Go (API Gateway -> Kinesis)
├── forwarder/                 # Lambda Kinesis -> Amplitude
├── ingestion/batch/           # Python: DATAtourisme, DVF (y opcional Inside Airbnb Bordeaux)
├── simulator/                 # generador de eventos en Python (anclado en IDs reales)
├── airflow/dags/              # DAGs (una responsabilidad por DAG)
├── dbt/                       # proyecto dbt (dbt-redshift)
├── ai/                        # enriquecimiento con LLM + harness de evaluación
├── infra/terraform/           # modules/ + envs/dev
├── metabase/                  # docker-compose + dashboards/preguntas exportados
├── observability/             # monitores/dashboards de Datadog como código (Terraform)
└── .github/workflows/         # CI/CD
```

## 5. Stack y convenciones

- **AWS**: S3, Lambda, API Gateway, Kinesis Data Streams + Firehose, EventBridge, SQS, Redshift Serverless,
  Secrets Manager, IAM, CloudWatch, Bedrock (módulo IA).
- **Terraform**: un módulo por componente en `infra/terraform/modules/`; estado remoto en S3 con locking;
  todos los recursos con tags `project`, `env`, `owner`, `cost-center`. `fmt`, `validate`, `tflint`, `checkov` en CI.
- **Python** 3.12, `uv` para dependencias, `ruff` (lint + format), `mypy --strict` en `ingestion/` y `ai/`, `pytest`.
- **Go** última versión estable, `golangci-lint`, tests table-driven, sin estado global en los handlers.
- **SQL / dbt**: `sqlfluff` (dialecto redshift, templater dbt). Capas:
  - `staging/stg_<fuente>__<entidad>`: 1:1 con raw; renombrar, castear, deduplicar; sin joins.
  - `intermediate/int_<tema>`: joins, sesionización, lógica de negocio.
  - `marts/`: `fct_events`, `fct_sessions`, `dim_listing` (SCD2), `dim_member`, `dim_commune`,
    `mart_sponsorship_funnel`, `mart_listing_trust_signals`, `mart_real_estate_prices`, `mart_pipeline_slo`.
  - Cada modelo tiene descripción, test de clave primaria (`unique` + `not_null`) y un owner en `meta`.
  - Las sources declaran `freshness`. Usar unit tests de dbt para la lógica no trivial.
- **Airflow** 3.x en local con Docker Compose (sin MWAA por defecto, por coste). DAGs idempotentes,
  parametrizados por fecha lógica, sin lógica de negocio dentro del fichero del DAG.
- **GitHub Actions**: autenticación con AWS por **OIDC** (sin claves estáticas). En cada PR: lint + tests
  unitarios + `dbt build` en un esquema `pr_<numero>` + `terraform plan` publicado como comentario.
  En main: despliegue.
- `pre-commit` replica los checks de la CI. Conventional Commits.

## 6. Flujo de trabajo: RePPIT (spec-driven, con validación humana)

Cada feature vive en `specs/<NNN-feature>/` y pasa por estas fases. **Detenerse en cada 🚦 y esperar la
aprobación explícita del usuario antes de continuar.**

1. **Re — Research** → `research.md`: estado actual del código, restricciones, documentación consultada
   (AWS, dbt, Airflow, DATAtourisme), opciones con pros y contras, preguntas abiertas. Sin código. 🚦
2. **P — Propose (Spec)** → `spec.md`: problema, alcance / fuera de alcance, contratos de datos, criterios de
   aceptación (verificables), requisitos no funcionales (latencia, tasa de error, coste), impacto RGPD. 🚦
3. **P — Plan** → `plan.md`: tareas pequeñas y ordenadas, ficheros afectados, tests por tarea, plan de rollback. 🚦
4. **I — Implement**: seguir el plan tarea a tarea; registrar desviaciones en `impl-log.md`. Tests primero
   cuando sea práctico.
5. **T — Test / Verify** → `verify.md`: cada criterio de aceptación con su evidencia (salida de tests,
   resultado de una query, captura, métrica). Después escribir `docs/interview/<feature>.md`: 5–10 preguntas
   probables de entrevista con respuesta, trade-offs y "qué haría a mayor escala".

Skills en `.claude/skills/` (crearlas según se necesiten): `reppit-research`, `reppit-spec`, `reppit-plan`,
`reppit-implement`, `reppit-verify`, `dbt-model`, `terraform-module`, `go-lambda`, `airflow-dag`,
`interview-notes`. Cargar la skill correspondiente antes de empezar cada fase.

## 7. Requisitos no funcionales (se miden, no se suponen)

Expuestos en `mart_pipeline_slo` y en Datadog:
- **Disponibilidad** del endpoint del collector (objetivo ≥ 99,5 % en las pruebas de carga de dev).
- **Tasa de error**: eventos inválidos / total, errores de Lambda, profundidad de la DLQ (alerta si > 0 durante 15 min).
- **Latencia**: p50/p95 de `loaded_at - event_ts` end-to-end; p95 del tiempo de respuesta del collector.
- **Freshness**: freshness de cada source en dbt; alerta si se incumple (DATAtourisme: < 26 h).
- **Coste**: alarma de AWS Budgets; estimación mensual por componente documentada en `docs/cost.md`.

## 8. Reglas RGPD (no negociables)

- `user_id`, emails, nombres → **seudonimización HMAC-SHA256** en el collector; la clave vive en Secrets Manager.
- DATAtourisme puede incluir contactos de profesionales (teléfono, email): se excluyen en la ingesta salvo
  necesidad justificada en una spec.
- Inside Airbnb `host_name`, `reviewer_name` → se eliminan o seudonimizan en la ingesta; nunca llegan a marts.
- Minimización: solo se aceptan los campos definidos en el tracking plan; los campos desconocidos se rechazan.
- Retención: lifecycle de S3 en raw (p. ej. 13 meses) y en quarantine (30 días).
- **Derecho de supresión**: tabla `gdpr_erasure_requests` + DAG de Airflow que borra en Redshift y reescribe
  las particiones de S3 afectadas; cada borrado queda auditado.
- IAM de mínimo privilegio por componente; Metabase usa un rol de solo lectura sobre `marts`.
- Todo texto libre que se envíe a un LLM se limpia de PII antes.

## 9. Módulo de IA (fase 2)

`ai/`: enriquecer anuncios (descripciones de DATAtourisme) y reseñas (Inside Airbnb Bordeaux) con un LLM vía
Bedrock: categoría, problemas de calidad, señales de anuncio sospechoso, sentimiento de reseñas.
Requisitos: prompts versionados, golden dataset etiquetado, evaluación offline (precision/recall, coste y
latencia por llamada) guardada en Redshift y visible en Metabase; resultados cacheados por hash de contenido;
nada de PII en los prompts.

## 10. Comandos

```
make setup            # uv sync, go mod download, pre-commit install
make data-download    # descarga DATAtourisme (44) + DVF (44) en ./data (en .gitignore)
make test             # pytest + go test + unit tests de dbt
make lint             # ruff, mypy, golangci-lint, sqlfluff, terraform fmt/validate, tflint
make up / make down   # Airflow + Metabase en local (docker compose)
make tf-plan ENV=dev  # terraform plan
make simulate RATE=50 # envía N eventos/s al collector
make dbt-build        # dbt build --target dev
make destroy ENV=dev  # elimina todos los recursos de AWS
```

## 11. Reglas para Claude

- **Nunca ejecutar `terraform apply`/`destroy` ni nada que cree recursos facturables en AWS sin confirmación
  explícita del usuario** en el mensaje actual. `plan` está permitido.
- Nunca commitear secretos, `.env`, datos raw ni el estado de Terraform. Usar `.env.example`.
- No saltarse ninguna 🚦 de RePPIT, ni siquiera en cambios "pequeños"; para arreglos triviales, una spec de 5 líneas.
- Cambios pequeños y revisables; una feature por rama `feat/<NNN-feature>`.
- Si hay dudas sobre el comportamiento de AWS, dbt, Airflow o DATAtourisme, consultar la documentación
  oficial y citarla en `research.md`.
- Preferir soluciones conocidas y aburridas; cualquier elección no obvia necesita un ADR.
- No hacer scraping de sitios web; solo fuentes abiertas o APIs con licencia.

## 12. Roadmap

- **M0** Arranque del repo: tooling, pre-commit, esqueleto de CI, backend de Terraform, alarma de presupuesto.
- **M1** Ingesta batch: DATAtourisme (44) + DVF (44) → S3 → Redshift raw → staging dbt (+ tests, freshness, snapshot SCD2).
- **M2** Tracking plan + collector Go + Kinesis/Firehose → S3, DLQ/quarantine.
- **M3** DAGs de Airflow: COPY incremental, dbt build, calidad de datos; marts (embudo, sesiones, señales de confianza, precios inmobiliarios).
- **M4** Dashboards en Metabase (embudo de padrinazgo, rendimiento de anuncios, mercado inmobiliario de Nantes, SLOs del pipeline) + forwarder a Amplitude.
- **M5** Observabilidad: monitores de Datadog como código, mart de SLOs, runbooks en `docs/runbooks/`.
- **M6** RGPD: auditoría de seudonimización, DAG de supresión, políticas de retención.
- **M7** Módulo de IA: enriquecimiento con LLM + harness de evaluación.

## Flujo de trabajo

El directorio `skills/` define un flujo de diseño → implementación (expuesto a Claude Code vía `.claude/skills → ../skills`). Las etapas escriben en `docs/design/<topic>/working/` (esto prevalece sobre la ruta por defecto de cada skill):

1. `reppit-research` → `current/research.md` (aquí, carpeta `current/research/`)
2. `reppit-propose` → `working/proposal.md`; el usuario elige en `## Decisión`
3. `reppit-plan` → `working/plan.md`
4. `reppit-implement` → ejecutar por fases
5. `reppit-test` → `working/test.md` con veredicto

Apoyos: `grill-me` (entrevistar para aclarar requisitos) y `adversarial-review` (revisar código o planes).
Para cambios no triviales (nuevas capas, modelo dimensional, decisiones de stack) seguir este flujo.

## Memoria de diseño (skill `design-memory`)

- **Empieza siempre por `docs/design/CONTEXT.md`** y para en cuanto la tarea esté clara. Orden: `CONTEXT.md` → `<topic>/context.md` → ficheros `always` de su mapa → solo los que la tarea nombre.
- No leas `CHANGELOG.md`, `evidence/`, `archive/` ni carpetas enteras (en especial `assets/research/salidas/`) salvo que la tarea nombre el fichero.
- Cada regla tiene un solo hogar: las de proyecto en este fichero o en `CONTEXT.md` § Conventions; las de un topic en `<topic>/context.md` § Rulings. En el resto, referenciar por nombre.
- Al cerrar una sesión que tocó `docs/design/`: CHANGELOG (una línea), reemplazar el bloque "Current state", actualizar el mapa y ejecutar `python3 skills/design-memory/scripts/design_check.py docs/design`. Un hook Stop (`.claude/settings.json`) lo ejecuta y devuelve los `FAIL`.

## Git

- Rama principal: `main`. Trabajar en ramas de feature; no hacer commit directo a `main`.