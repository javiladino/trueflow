---
type: Research
date: 2026-11-05
scope: Mapeo de arquitectura y datos de la propuesta `trueflow` (CLAUDE.md)
---

# RESEARCH — Mapeo de arquitectura y datos

Modo read-only: solo se documenta lo que existe hoy. Este fichero es el único artefacto creado.

## 1. Estado actual del repositorio

```
$ find . -type f -not -path '*/.git/*' | grep -v DS_Store
./CLAUDE.md
./skills/{reppit-research,reppit-propose,reppit-plan,reppit-implement,reppit-test,
          adversarial-review,grill-me,design-memory}/...   (SKILL.md + plantillas/scripts)

$ git status
fatal: no es un repositorio git (ni ninguno de los directorios superiores): .git

$ ls -la .claude ; ls docs
ls: .claude: No such file or directory
ls: docs: No such file or directory

$ python3 skills/design-memory/scripts/design_check.py docs/design
usage error: docs/design/CONTEXT.md not found

$ wc -l CLAUDE.md
223 CLAUDE.md
```

**Hallazgo:** el proyecto es solo una propuesta. No existe ninguno de los directorios de §4 de CLAUDE.md
(`collector/`, `dbt/`, `infra/terraform/`, `tracking-plan/`, `airflow/`, `Makefile`, etc.). No hay
repositorio git, ni `.claude/`, ni `docs/design/`.

## 2. Tooling local disponible

```
python3: Python 3.9.6          (CLAUDE.md §5 exige 3.12 → lo gestionaría uv)
uv:      0.13.5
go:      not found             (requerido por collector/ y forwarder/)
terraform: not found           (requerido por infra/ y observability/)
aws:     not found
dbt:     dbt-core 1.10.23 (instalado vía uv tool sobre Python 3.9; sin adapter redshift listado)
docker:  29.8.2                (Airflow + Metabase local)
make:    GNU Make 3.81
```

## 3. Arquitectura propuesta (CLAUDE.md §3)

| Capa | Componentes | Lenguaje / servicio |
|---|---|---|
| Captura de eventos | API Gateway → Lambda collector (validación JSON Schema, enriquecimiento, HMAC de PII) | Go |
| Streaming | Kinesis Data Stream → Firehose → S3 raw `events/dt=/hour=/` | AWS gestionado |
| Errores | S3 quarantine (30 d) + SQS DLQ | AWS |
| Reenvío | Lambda forwarder Kinesis → Amplitude HTTP API (lotes) | — |
| Batch | EventBridge Scheduler → Lambda ingest (DATAtourisme diario, DVF) → S3 raw | Python |
| Orquestación | Airflow 3.x local (Docker): COPY → `raw`, dbt build, calidad, borrado RGPD | Python |
| Warehouse | Redshift Serverless eu-west-3; dbt staging → intermediate → marts | SQL/dbt |
| Consumo | Metabase (rol read-only sobre `marts`), Datadog, módulo IA (Bedrock) | — |

### Flujo de datos

1. **Eventos (tiempo casi real):** simulador → collector → válidos a Kinesis (fan-out a Firehose/S3 y a
   Amplitude), inválidos a quarantine/DLQ. Entrega at-least-once → dedupe por `event_id` en staging.
2. **Batch (diario):** DATAtourisme (alojamientos/POI 44) y DVF (transacciones 44) → S3 raw.
3. **ELT:** Airflow ejecuta COPY idempotente con manifest a `raw`, después `dbt build`.
   Eventos tardíos: incrementales con lookback sobre `received_at`. `dim_listing` en SCD2 (snapshot).
4. **Consumo:** marts (`fct_events`, `fct_sessions`, `dim_listing`, `dim_member`, `dim_commune`,
   `mart_sponsorship_funnel`, `mart_listing_trust_signals`, `mart_real_estate_prices`, `mart_pipeline_slo`).

## 4. Caso de uso y datos

- Marketplace por padrinazgo; verticales: alquiler de temporada, inmobiliario, servicios; ámbito Nantes / 44.
- Fuentes reales: DATAtourisme (Licence Ouverte 2.0, API con clave), DVF/geo-DVF (ficheros por dpto.),
  Inside Airbnb Bordeaux (opcional, solo texto de reseñas, fase 2). Eventos de producto: sintéticos
  anclados en IDs reales.
- 9 eventos del tracking plan: `signup_started`, `sponsorship_requested`, `sponsorship_accepted`,
  `member_validated`, `listing_viewed`, `listing_published`, `contact_requested`, `booking_requested`,
  `review_posted`. Aún no existe `tracking-plan/`.
- RGPD (§8): HMAC-SHA256 en collector, exclusión de contactos profesionales DATAtourisme, rechazo de
  campos desconocidos, retención S3 13 meses / 30 días, DAG de supresión auditado.
- NFR (§7): disponibilidad ≥ 99,5 %, DLQ > 0 durante 15 min, p50/p95 latencia, freshness DATAtourisme < 26 h,
  AWS Budgets.

## 5. Inconsistencias observadas en CLAUDE.md

Se documentan tal como están; no se proponen cambios.

```
$ grep -n "reppit-" CLAUDE.md
131: Skills en `.claude/skills/` ...: `reppit-research`, `reppit-spec`, `reppit-plan`,
132: `reppit-implement`, `reppit-verify`, ...
206: 1. `reppit-research` → `current/research.md` (aquí, carpeta `current/research/`)
207: 2. `reppit-propose` → `working/proposal.md`
210: 5. `reppit-test` → `working/test.md`
```

1. **Dos flujos de trabajo:** §6 usa `specs/<NNN>/{research,spec,plan,impl-log,verify}.md`; la sección
   "Flujo de trabajo" usa `docs/design/<topic>/working/`. Esta última dice prevalecer.
2. **Nombres de skills:** §6 cita `reppit-spec`/`reppit-verify`; en `skills/` existen `reppit-propose`/`reppit-test`.
   Las skills de dominio (`dbt-model`, `terraform-module`, `go-lambda`, `airflow-dag`, `interview-notes`) no existen.
3. **Ruta de research:** línea 206 dice `current/research.md`; el SKILL.md dice `working/research.md`.
4. **`.claude/skills → ../skills`:** el symlink no existe; tampoco `.claude/settings.json` (hook Stop citado).
5. **Git:** CLAUDE.md asume ramas `feat/<NNN>` y `main`, pero el directorio no es un repo git.
6. **Este fichero** (`docs/RESEARCH.md`) se pidió fuera de la convención `docs/design/` .

## 6. Preguntas abiertas

- ~~¿Qué convención de rutas manda?~~ Resuelto 2026-11-05: manda `docs/design/`. Git inicializado (`main`) y symlink `.claude/skills → ../skills` creado.
- ¿Se instalan Go, Terraform y AWS CLI localmente o solo en CI?
- ¿Python 3.12 vía `uv python install`? ¿dbt-redshift en el proyecto (no como tool global sobre 3.9)?
- ¿Acceso a la API de DATAtourisme (clave) o exports de data.gouv.fr para M1?
