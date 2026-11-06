# CLAUDE.md — Plataforma de datos para un marketplace de confianza (proyecto portfolio)

## 1. Propósito

Plataforma de datos end-to-end para un **marketplace de anuncios basado en la confianza** (los miembros
entran por padrinazgo / "parrainage"; verticales principales: alquiler de temporada, inmobiliario, servicios).
Ámbito geográfico: **Nantes y Loire-Atlantique (44)**.

Es un proyecto **educativo** de preparación para una entrevista de Data Engineer: cada decisión técnica debe ser
**explicable, medida y documentada**, no solo funcionar. Vive unas semanas en una cuenta AWS en **Free plan** y
se desmantela al terminar (ADR 0001).

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

## 3. Arquitectura — lakehouse en S3 sin streaming (ADR 0001)

```
[Simulador / clientes] --HTTPS--> API Gateway --> Lambda collector (Go)
        - validación JSON Schema (tracking plan)  - enriquecimiento  - seudonimización de PII
        - agrupa eventos por invocación y escribe con PutObject
              |  válidos                                  | inválidos
              v                                           v
   S3 raw  s3://<bucket>/events/dt=YYYY-MM-DD/hour=HH/   S3 quarantine + SQS DLQ
              |
              +--(S3 ObjectCreated)--> Lambda forwarder --> Amplitude HTTP API (por lotes)
              |
EventBridge Scheduler --> Lambda batch ingest (DATAtourisme diario, DVF) --> S3 raw
                                  |
                       Airflow (orquestación, local)
                        - registro de particiones en Glue Data Catalog (idempotente)
                        - dbt build con dbt-athena (staging -> intermediate -> marts, tablas Iceberg) + tests
                        - compactación Iceberg, checks de calidad / freshness, job de borrado RGPD
                                  |
                  Amazon Athena + Glue Data Catalog + Iceberg en S3 (eu-west-3)
                                  |
              Metabase (driver Athena)   Datadog (monitores del pipeline)   Módulo IA (tablas de evaluación)
```

Decisiones clave (cualquier cambio se registra como ADR en `docs/adr/`):
- Región **eu-west-3 (París)**: argumento de residencia de datos para el RGPD.
- **Solo servicios disponibles en el Free plan**: sin Kinesis Data Streams, Firehose ni Redshift (ADR 0001;
  hechos en `docs/design/findings.md`).
- Collector en **Go** (cold start bajo, validación tipada); batch y orquestación en **Python**.
- **ELT**: se aterriza en raw (S3, inmutable y re-procesable) y se transforma con dbt sobre Athena.
- Entrega at-least-once → **deduplicación por `event_id`** en staging de dbt.
- Eventos tardíos: modelos incrementales con ventana de lookback sobre `received_at`.
- Sin buffer de streaming → el collector agrupa eventos y Airflow compacta las tablas Iceberg (ficheros pequeños).
- DATAtourisme cambia a diario → snapshot dbt (**SCD2**) de `dim_listing` para historizar cambios.

## 4. Estructura del repositorio

```
.
├── CLAUDE.md
├── skills/                    # skills RePPIT + design-memory (expuestas como .claude/skills → ../skills)
├── docs/design/               # memoria de diseño: CONTEXT, README, CHANGELOG, findings, un topic por feature (§6)
├── docs/adr/                  # Architecture Decision Records (NNNN-titulo.md)
├── docs/architecture/         # diagramas Mermaid de arquitectura y flujo de datos
├── docs/interview/            # preguntas/respuestas y trade-offs por feature
├── docs/runbooks/             # qué hacer cuando salta cada alerta
├── tracking-plan/             # JSON Schemas de eventos + changelog
├── collector/                 # Lambda Go (API Gateway -> S3)
├── forwarder/                 # Lambda S3 ObjectCreated -> Amplitude
├── ingestion/batch/           # Python: DATAtourisme, DVF (y opcional Inside Airbnb Bordeaux)
├── simulator/                 # generador de eventos en Python (anclado en IDs reales)
├── airflow/dags/              # DAGs (una responsabilidad por DAG)
├── dbt/                       # proyecto dbt (dbt-athena, tablas Iceberg)
├── ai/                        # enriquecimiento con LLM + harness de evaluación
├── infra/terraform/           # bootstrap/ + modules/ + envs/dev
├── metabase/                  # docker-compose + dashboards/preguntas exportados
├── observability/             # monitores/dashboards de Datadog como código (Terraform)
└── .github/workflows/         # CI/CD
```

## 5. Stack y convenciones

- **AWS**: S3, Lambda, API Gateway, EventBridge, SQS, Athena, Glue Data Catalog, Secrets Manager, IAM,
  CloudWatch, Budgets, Bedrock (módulo IA, disponibilidad en el Free plan por verificar).
- **Terraform**: un módulo por componente en `infra/terraform/modules/`; estado remoto en S3 con
  `use_lockfile` (sin DynamoDB); todos los recursos con tags `project`, `env`, `owner`, `cost-center`.
  `fmt`, `validate`, `tflint`, `checkov` en CI.
- **Python** 3.12, `uv` para dependencias, `ruff` (lint + format), `mypy --strict` en `ingestion/` y `ai/`, `pytest`.
- **Go** última versión estable, `golangci-lint`, tests table-driven, sin estado global en los handlers.
- Versiones fijadas: `docs/design/m0-foundations/working/research.md` §15 (hasta que el plan de M0 las fije en código).
- **SQL / dbt**: `dbt-athena` con tablas Iceberg; `sqlfluff` (dialecto athena, templater dbt). Capas:
  - `staging/stg_<fuente>__<entidad>`: 1:1 con raw; renombrar, castear, deduplicar; sin joins.
  - `intermediate/int_<tema>`: joins, sesionización, lógica de negocio.
  - `marts/`: `fct_events`, `fct_sessions`, `dim_listing` (SCD2), `dim_member`, `dim_commune`,
    `mart_sponsorship_funnel`, `mart_listing_trust_signals`, `mart_real_estate_prices`, `mart_pipeline_slo`.
  - Cada modelo tiene descripción, test de clave primaria (`unique` + `not_null`) y un owner en `meta`.
  - Las sources declaran `freshness`. Usar unit tests de dbt para la lógica no trivial.
- **Airflow** 3.x en local con Docker Compose (sin MWAA, por coste). DAGs idempotentes,
  parametrizados por fecha lógica, sin lógica de negocio dentro del fichero del DAG.
- **GitHub Actions**: autenticación con AWS por **OIDC** (sin claves estáticas; el `sub` del repo usa IDs
  inmutables, ver `docs/design/findings.md`). En cada PR: lint + tests unitarios + `dbt build` en una base de
  Glue `pr_<numero>` + `terraform plan` publicado como comentario. En main: despliegue. Los checks de la CI son
  obligatorios en `main`.
- `pre-commit` replica los checks de la CI. Conventional Commits.

## 6. Flujo de trabajo: RePPIT (con validación humana)

Cada feature es un topic en `docs/design/<topic>/` (convención "Raíz de diseño" en `docs/design/CONTEXT.md`) y
pasa por estas fases. Cargar la skill de la fase antes de empezarla. **Detenerse en cada 🚦 y esperar la
aprobación explícita del usuario antes de continuar.**

1. **Re — Research** (`reppit-research`) → `working/research.md`: estado actual del código, restricciones,
   documentación oficial consultada y citada (AWS, dbt, Airflow, DATAtourisme), opciones con pros y contras,
   preguntas abiertas. Sin código. 🚦
2. **P — Propose** (`reppit-propose`) → `working/proposal.md`: 2–3 propuestas con trade-offs, criterios de
   aceptación verificables, requisitos no funcionales (latencia, tasa de error, coste) e impacto RGPD; el usuario
   elige en `## Decision`. 🚦
3. **P — Plan** (`reppit-plan`) → `working/plan.md`: tareas pequeñas y ordenadas, ficheros afectados, tests por
   tarea, plan de rollback. 🚦
4. **I — Implement** (`reppit-implement`): seguir el plan fase a fase; las desviaciones van al Journal de
   `<topic>/context.md`. Tests primero cuando sea práctico.
5. **T — Test** (`reppit-test`) → `working/test.md`: cada criterio de aceptación con su evidencia y un veredicto.
   Después escribir `docs/interview/<topic>.md`: 5–10 preguntas probables de entrevista con respuesta,
   trade-offs y "qué haría a mayor escala".

Apoyos: `grill-me` (entrevistar para aclarar requisitos, una pregunta cada vez) y `adversarial-review`
(revisar código o planes). Skills de dominio previstas, aún sin crear: `dbt-model`, `terraform-module`,
`go-lambda`, `airflow-dag`, `interview-notes`.

## 7. Memoria de diseño (skill `design-memory`)

`docs/design/` es la memoria entre sesiones. Cargar `design-memory` siempre que se lea o escriba ahí.

- **Orden de lectura** (parar en cuanto la tarea esté clara): `CONTEXT.md` → `<topic>/context.md` → ficheros
  `always` de su mapa de documentos → solo los que la tarea nombre. `findings.md` solo si la tarea toca uno de
  sus temas.
- No leer `CHANGELOG.md`, `evidence/`, `archive/` ni carpetas enteras salvo que la tarea nombre el fichero.
- **Dónde va cada cosa:** lo que es verdad ahora → `CONTEXT.md` (un solo bloque "Current state", se reemplaza);
  lo que pasó → `CHANGELOG.md`; hechos duraderos que otros topics necesitan → `findings.md`; lo de una sesión en
  un topic → Journal de `<topic>/context.md` (6–8 bloques como máximo).
- **Cada regla tiene un solo hogar:** las reglas estables del proyecto en este fichero; las decisiones de
  proyecto en `CONTEXT.md` § Conventions; las de un topic en `<topic>/context.md` § Rulings. En el resto,
  referenciar por nombre.
- **Carpetas por ciclo de vida:** `working/` (borradores RePPIT), `current/` (entregables vigentes),
  `evidence/` (material externo, inmutable), `archive/` (superado). Al cerrar un ciclo (test en PASS y
  aceptado): decisiones a su hogar, entregable a `current/`, borradores a `archive/` con `git mv`.
- **Cierre de sesión** que tocó `docs/design/`: una línea en CHANGELOG, reemplazar "Current state", actualizar el
  mapa, añadir findings si los hay y ejecutar `python3 skills/design-memory/scripts/design_check.py docs/design`
  hasta 0 FAIL. El hook Stop de `.claude/settings.json` lo ejecuta y devuelve los `FAIL`.

## 8. Requisitos no funcionales (se miden, no se suponen)

Expuestos en `mart_pipeline_slo` y en Datadog:
- **Disponibilidad** del endpoint del collector (objetivo ≥ 99,5 % en las pruebas de carga de dev).
- **Tasa de error**: eventos inválidos / total, errores de Lambda, profundidad de la DLQ (alerta si > 0 durante 15 min).
- **Latencia**: p50/p95 de `loaded_at - event_ts` end-to-end; p95 del tiempo de respuesta del collector.
- **Freshness**: freshness de cada source en dbt; alerta si se incumple (DATAtourisme: < 26 h).
- **Coste**: AWS Budgets de **25 USD/mes** con alertas por email (el email va en un `*.tfvars` local, nunca en
  el repo); estimación mensual por componente en `docs/cost.md`; bytes escaneados por Athena por query.

## 9. Reglas RGPD (no negociables)

- `user_id`, emails, nombres → **seudonimización HMAC-SHA256** en el collector; la clave vive en Secrets Manager.
- DATAtourisme puede incluir contactos de profesionales (teléfono, email): se excluyen en la ingesta salvo
  necesidad justificada en una propuesta.
- Inside Airbnb `host_name`, `reviewer_name` → se eliminan o seudonimizan en la ingesta; nunca llegan a marts.
- Minimización: solo se aceptan los campos definidos en el tracking plan; los campos desconocidos se rechazan.
- Retención: lifecycle de S3 en raw (p. ej. 13 meses) y en quarantine (30 días).
- **Derecho de supresión**: tabla `gdpr_erasure_requests` + DAG de Airflow que hace `DELETE` en las tablas
  Iceberg vía Athena (más expiración de snapshots) y reescribe los objetos raw de S3 afectados; cada borrado
  queda auditado.
- IAM de mínimo privilegio por componente; Metabase usa un rol de solo lectura sobre la base de Glue `marts`.
- Todo texto libre que se envíe a un LLM se limpia de PII antes.

## 10. Módulo de IA (fase 2)

`ai/`: enriquecer anuncios (descripciones de DATAtourisme) y reseñas (Inside Airbnb Bordeaux) con un LLM vía
Bedrock: categoría, problemas de calidad, señales de anuncio sospechoso, sentimiento de reseñas.
Requisitos: prompts versionados, golden dataset etiquetado, evaluación offline (precision/recall, coste y
latencia por llamada) guardada en tablas Iceberg y visible en Metabase; resultados cacheados por hash de
contenido; nada de PII en los prompts.

## 11. Comandos

```
make setup            # uv sync, go mod download, pre-commit install
make data-download    # descarga DATAtourisme (44) + DVF (44) en ./data (en .gitignore)
make test             # pytest + go test + unit tests de dbt
make lint             # ruff, mypy, golangci-lint, sqlfluff, terraform fmt/validate, tflint, checkov
make up / make down   # Airflow + Metabase en local (docker compose)
make tf-plan ENV=dev  # terraform plan
make simulate RATE=50 # envía N eventos/s al collector
make dbt-build        # dbt build --target dev
make destroy ENV=dev  # elimina todos los recursos de AWS (desmantelamiento final)
```

## 12. Reglas para Claude

- **Nunca ejecutar `terraform apply`/`destroy` ni nada que cree recursos facturables en AWS sin confirmación
  explícita del usuario** en el mensaje actual. `plan` está permitido.
- No usar servicios que exijan Paid plan (p. ej. Kinesis, Firehose, Redshift) sin un ADR aprobado.
- Nunca commitear secretos, `.env`, `*.tfvars`, datos raw ni el estado de Terraform. Usar `.env.example`.
  El repo es público.
- No saltarse ninguna 🚦 de RePPIT, ni siquiera en cambios "pequeños"; para arreglos triviales, una propuesta de 5 líneas.
- Cambios pequeños y revisables; una feature por rama `feat/<NNN-feature>`; nunca commit directo a `main`
  (protegida: solo por PR).
- Si hay dudas sobre el comportamiento de AWS, dbt, Airflow o DATAtourisme, consultar la documentación
  oficial y citarla en `research.md`.
- Preferir soluciones conocidas y aburridas; cualquier elección no obvia necesita un ADR.
- No hacer scraping de sitios web; solo fuentes abiertas o APIs con licencia.

## 13. Roadmap

- **M0** Arranque del repo: tooling, pre-commit, esqueleto de CI, backend de Terraform, alarma de presupuesto.
- **M1** Ingesta batch: DATAtourisme (44) + DVF (44) → S3 raw → Glue/Athena → staging dbt (+ tests, freshness, snapshot SCD2).
- **M2** Tracking plan + collector Go → S3 raw (escritura directa por lotes), DLQ/quarantine.
- **M3** DAGs de Airflow: registro de particiones, dbt build, compactación Iceberg, calidad de datos; marts (embudo, sesiones, señales de confianza, precios inmobiliarios).
- **M4** Dashboards en Metabase (embudo de padrinazgo, rendimiento de anuncios, mercado inmobiliario de Nantes, SLOs del pipeline) + forwarder a Amplitude.
- **M5** Observabilidad: monitores de Datadog como código, mart de SLOs, runbooks en `docs/runbooks/`.
- **M6** RGPD: auditoría de seudonimización, DAG de supresión, políticas de retención.
- **M7** Módulo de IA: enriquecimiento con LLM + harness de evaluación.
- **Cierre** Desmantelamiento: `make destroy ENV=dev`, vaciado de buckets y comprobación de coste 0.
