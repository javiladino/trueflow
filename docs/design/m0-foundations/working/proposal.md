---
type: Proposal
---

# m0-foundations: Solution Proposals

> **Task:** elegir la arquitectura de plataforma que M0 debe preparar (cuenta, backend de estado, CI, presupuesto) dentro del coste disponible.
> **Date:** 2026-11-05
> **Research source:** `docs/design/m0-foundations/working/research.md` §3 (arquitectura), §8–12 (M0), §15 (respuestas y versiones)
> **Review:** el agente `adversarial-reviewer` no existe en este entorno; la ronda adversarial la hice yo en esta sesión, con `grill-me` para cuestionar cada opción (ver § Grill-me).
> **Ruta:** el usuario pidió `PROPOSAL.md`; se guarda aquí por la convención "Raíz de diseño" (CONTEXT § Conventions).

## Context

**Request:** proponer 2–3 arquitecturas distintas para desarrollar el proyecto, evaluarlas y recomendar una; M0 deja preparada la que se elija.

**Success criteria (del usuario):** simplicidad · mantenibilidad · rendimiento · impacto en las fases posteriores.
**Restricciones añadidas por el usuario:** presupuesto de 25 USD/mes (research §15.2); cuenta AWS `437266392155` en **Free plan** con 170 USD de crédito.

**Key constraints from the research and from the official docs consulted today:**
- La arquitectura de CLAUDE.md usa Kinesis Data Streams + Firehose y Redshift Serverless en eu-west-3 (research §3).
- **Free plan** ([AWS docs](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/free-tier-plans.html)): "ends after six months or when your credits are fully used"; después "your account closes automatically". Solo da "access to select AWS services"; el paso a Paid plan conserva el crédito restante ("automatically applied to future AWS bills").
- **Kinesis Data Streams** "is NOT currently available in AWS Free Tier" ([pricing](https://aws.amazon.com/kinesis/data-streams/pricing/)). Ejemplos en US-East: shard provisionado $0.015/h (≈ 11 USD/mes por shard 24/7); on-demand ≈ $0.040 por stream-hora (≈ 29 USD/mes).
- **Redshift Serverless** ([pricing](https://aws.amazon.com/redshift/pricing/)): "begins at $1.50 per hour", base de 4 a 1024 RPU, facturación por segundo con mínimo de 60 s. Hay una prueba con **300 USD de crédito durante 90 días** para cuentas que nunca lo usaron, y "is not part of the AWS Free Tier" ([free trial](https://aws.amazon.com/redshift/free-trial/)).
- **Firehose** cobra por GB en incrementos de 5 KB por registro ([pricing](https://aws.amazon.com/firehose/pricing/), ejemplo $0.029/GB).
- No hay precios verificados para eu-west-3 (las páginas solo mostraron US-East); las cifras de abajo son **estimaciones** con esos precios.
- Común a las tres propuestas (research §10–12): backend S3 con `use_lockfile`, OIDC con `sub` inmutable `repo:javiladino@97758532/trueflow@1447219459`, `aws_budgets_budget` de 25 USD, Terraform 1.17.1 / provider AWS 6.72.0.

## Proposal 1: AWS completo según CLAUDE.md, con disciplina de coste

### Overview
Se mantiene la arquitectura de CLAUDE.md §3 tal cual. La cuenta pasa a Paid plan (el crédito se conserva) y el coste se controla con la prueba de Redshift Serverless (300 USD / 90 días), un único shard provisionado de Kinesis, base de 4 RPU y entornos que se destruyen cuando no se usan.

### Key Changes
| Component | Change |
|-----------|--------|
| Cuenta AWS | Paso a Paid plan; activar la prueba de Redshift Serverless al llegar a M1 |
| `infra/terraform/bootstrap/` | New: bucket de estado (estado local, después migrado) |
| `infra/terraform/envs/dev/` | New: backend S3 `use_lockfile`, budget de 25 USD, rol OIDC |
| `infra/terraform/modules/streaming` | New (M2): Kinesis provisionado con 1 shard y variable `enable_streaming` para apagarlo |
| `infra/terraform/modules/warehouse` | New (M1): Redshift Serverless con base de 4 RPU |
| CLAUDE.md | No change |

### Trade-offs
| Benefit | Risk |
|---------|------|
| Cero ADRs de desvío: coincide con CLAUDE.md §3 y con el relato de la entrevista (Kinesis, Redshift, COPY con manifest) | Coste fijo sin prueba: Kinesis ≈ 11 USD + Redshift ≈ 1.50 USD/h activa → 30 min/día ≈ 23 USD; **supera los 25 USD/mes** |
| La prueba de 300 USD cubre Redshift 90 días si la cuenta es elegible | La prueba caduca a los 90 días y M1–M5 pueden durar más |
| Rendimiento real de warehouse columnar y streaming con orden por shard | Requiere `terraform destroy` disciplinado; un olvido cuesta ≈ 36 USD/día con Redshift activo |
| Paid plan conserva los 170 USD de crédito | Paid plan quita el cierre automático: sin budget y alertas, el gasto ya no tiene tope |

### Validation
1. `aws budgets describe-budgets --account-id 437266392155` muestra un budget de 25 USD con alertas.
2. `aws redshift-serverless list-workgroups` tras M1: `baseCapacity` = 4.
3. `terraform plan -var enable_streaming=false` en `envs/dev` no contiene `aws_kinesis_stream`.
4. Coste diario en Cost Explorer < 0.85 USD (25/30) en días sin pruebas de carga.

### Open Questions
1. ¿La cuenta es elegible para la prueba de Redshift Serverless? Si no lo es, esta propuesta no cabe en el presupuesto.
2. Precios de eu-west-3 sin verificar; pueden ser algo más altos que los de US-East.

## Proposal 2: Lakehouse serverless en S3 (Athena + Iceberg)

### Overview
Todo es pago por uso: el collector escribe en Firehose Direct PUT (sin Kinesis Data Streams), los datos se guardan en S3 como tablas Iceberg y dbt corre con el adapter de Athena en lugar de Redshift. El coste fijo es casi cero.

### Key Changes
| Component | Change |
|-----------|--------|
| Ingesta de eventos | Modified: collector → Firehose Direct PUT → S3; desaparece Kinesis Data Streams |
| Forwarder a Amplitude | Modified: se dispara por eventos de S3 o SQS, ya no consume del stream |
| Warehouse | Modified: Athena + Glue Data Catalog + Iceberg en lugar de Redshift Serverless; `dbt-athena` en lugar de `dbt-redshift` |
| Airflow | Modified: el paso "COPY a raw" se convierte en registrar particiones / `MSCK` o en tablas externas |
| CLAUDE.md §3, §5 y docs/adr | Modified: requiere ADR (cambio de warehouse y de streaming) |

### Trade-offs
| Benefit | Risk |
|---------|------|
| Coste estimado de unos pocos USD/mes con volúmenes de dev; cabe en 25 USD sin destruir nada | Desvía la arquitectura de CLAUDE.md; hay que reescribir §3, §5 y el roadmap M1–M3 |
| Sin servidores que apagar; mantenimiento bajo | Sin streaming: se pierde el fan-out de Kinesis y el orden por shard (CLAUDE.md §3) |
| Iceberg da MERGE y borrado por fila, útil para la supresión RGPD (CLAUDE.md §8) | Latencia de consultas de segundos y cobro por TB escaneado: los dashboards de Metabase pagan cada refresco |
| Raw inmutable en S3 se mantiene (ELT) | No verificado aún: soporte de `dbt snapshot` (SCD2) y de unit tests en `dbt-athena` |

### Validation
1. `uv pip install dbt-athena` y `dbt debug` contra el workgroup de Athena.
2. Un `dbt snapshot` sobre una tabla Iceberg de prueba crea las columnas `dbt_valid_from` y `dbt_valid_to`.
3. `aws firehose describe-delivery-stream` muestra el destino `s3://<bucket>/events/` con prefijo `dt=/hour=`.
4. Coste mensual en Cost Explorer < 5 USD.

### Open Questions
1. ¿Están Athena y Firehose disponibles en el Free plan? (la página de restricciones no cargó).
2. ¿Acepta el usuario que la historia de entrevista pase de "Redshift + Kinesis" a "lakehouse"?

## Proposal 3: Local-first con AWS mínimo

### Overview
En AWS solo queda lo barato y demostrable: API Gateway, el Lambda collector, S3, Secrets Manager, el budget y OIDC. El warehouse, la orquestación y los dashboards corren en local con Docker: DuckDB + `dbt-duckdb` leyendo de S3, Airflow y Metabase.

### Key Changes
| Component | Change |
|-----------|--------|
| Ingesta de eventos | Modified: el collector escribe lotes directos a S3 (sin Kinesis ni Firehose) |
| Warehouse | Modified: DuckDB local (lee Parquet/JSON de S3 vía `httpfs`) con `dbt-duckdb` |
| Airflow, Metabase | No change: ya eran locales (CLAUDE.md §5) |
| CLAUDE.md §3, §5 y docs/adr | Modified: requiere ADR |

### Trade-offs
| Benefit | Risk |
|---------|------|
| Lo más simple: casi todo cabe en el Free plan y el coste esperado es ~0–2 USD/mes | El warehouse no es gestionado ni compartible; Metabase no tiene un endpoint seguro en la nube |
| Ciclo de desarrollo rápido y tests de dbt sin conexión | Se pierden Redshift, Kinesis y Firehose: es la parte más "AWS" del relato de entrevista |
| DuckDB soporta snapshots y unit tests de dbt | Escalabilidad limitada a una máquina; el job RGPD tiene que reescribir ficheros a mano |
| El Free plan sigue siendo válido 6 meses | La cuenta se cierra a los 6 meses si no se pasa a Paid plan |

### Validation
1. `dbt build --target dev` con `dbt-duckdb` lee un fichero de `s3://<bucket>/events/`.
2. `curl -X POST <api>/events` → 202 y un objeto nuevo en S3 en menos de 60 s.
3. Coste mensual en Cost Explorer < 2 USD.

### Open Questions
1. ¿Es aceptable un warehouse no gestionado para la demo ante un entrevistador?

## Grill-me (cuestionamiento de cada opción)

- **P1 — caso borde:** si la prueba de Redshift caduca a mitad de M3, el coste salta a ≈ 23 USD/mes solo de cómputo con 30 min/día. Mitigación: hacer las demos dentro de los 90 días o pausar el workgroup; aun así, el riesgo es real.
- **P1 — escalabilidad:** es la única que escala sin rediseño (más shards, más RPU).
- **P2 — manejo de tipos:** Athena e Iceberg tienen otro sistema de tipos (sin `SUPER` de Redshift para JSON semiestructurado); los eventos con propiedades variables necesitan `json_extract` o un esquema rígido. Los modelos staging cambian.
- **P2 — casos borde:** at-least-once con Firehose Direct PUT sigue siendo cierto, así que la dedupe por `event_id` se mantiene; los eventos tardíos con lookback funcionan igual con particiones `dt`.
- **P3 — escalabilidad:** DuckDB local no tiene concurrencia de escritura; Airflow y Metabase compiten por el mismo fichero `.duckdb`.
- **P3 — RGPD:** borrar en Parquet de S3 exige reescribir ficheros, sin MERGE; es el escenario más frágil para el DAG de supresión.
- **Todas — cuenta:** en el Free plan la cuenta se cierra a los 6 meses, así que las tres necesitan pasar a Paid plan antes de terminar el roadmap (M7).

### Respuestas del usuario (grill-me)

- Plan de la cuenta AWS → seguir en Free plan (decisión del usuario, 2026-11-05). Consecuencia: P1 queda descartada porque Kinesis Data Streams no está en el Free Tier y la prueba de Redshift "is not part of the AWS Free Tier"; quedan P2 (si Athena y Firehose están permitidos) y P3. La cuenta se cierra en 6 meses como máximo.
- Vida del proyecto → unas semanas; se desmantela al terminar (usuario, 2026-11-06). Consecuencia: el cierre del Free plan a los 6 meses deja de ser un riesgo.
- Disponibilidad en el Free plan → comprobada por el usuario en la consola (2026-11-06): **Athena y Glue abren; Amazon Data Firehose exige pasar a Paid plan**. Consecuencia: P2 solo es viable sin Firehose (variante P2b).

## Proposal 2b: Lakehouse en S3 sin Firehose (variante de P2 para el Free plan)

### Overview
Igual que P2 (Athena + Glue Data Catalog + Iceberg + `dbt-athena`), pero el collector escribe directamente en S3 en lugar de pasar por Firehose. Usa solo servicios que el usuario ha visto abiertos en el Free plan, más Lambda, API Gateway y S3.

### Key Changes
| Component | Change |
|-----------|--------|
| Collector (Go) | Modified: escribe objetos JSON por lotes en `s3://<bucket>/events/dt=YYYY-MM-DD/hour=HH/` con `PutObject` |
| Kinesis + Firehose | Removed |
| Forwarder a Amplitude | Modified: se dispara con notificaciones de S3 (`s3:ObjectCreated`) |
| Warehouse | Athena + Glue + Iceberg, `dbt-athena` (como P2) |
| CLAUDE.md §3, §5 y docs/adr | Modified: ADR por cambio de streaming y de warehouse |

### Trade-offs
| Benefit | Risk |
|---------|------|
| Cabe en el Free plan sin upgrade (comprobación del usuario en la consola) | Sin buffer de streaming: cada invocación escribe en S3; hay que agrupar eventos en el collector o crear muchos objetos pequeños |
| Coste de céntimos (Athena, ejemplo de 5 USD/TB escaneado) | Muchos ficheros pequeños hacen más lentas las consultas de Athena; hace falta compactar (Iceberg `OPTIMIZE`) |
| Iceberg da MERGE y DELETE por fila para la supresión RGPD | Mismos puntos sin verificar que P2: `dbt snapshot` y unit tests en `dbt-athena` |
| Desmontaje simple: `terraform destroy` + vaciar buckets | Se pierde la parte de Kinesis del relato; hay que contarlo como trade-off de coste en el ADR |

### Validation
1. `curl -X POST <api>/events` → 202 y un objeto nuevo bajo `events/dt=.../hour=.../`.
2. `dbt debug` con `dbt-athena` contra el workgroup; `dbt snapshot` sobre Iceberg crea `dbt_valid_from`/`dbt_valid_to`.
3. Una query de Athena sobre `raw.events` devuelve el evento enviado; el `event_id` duplicado aparece una sola vez en staging.
4. Cost Explorer < 5 USD/mes.

## Recommendation

| Criteria | P1 AWS completo | P2 Lakehouse | P3 Local-first |
|----------|-----------------|--------------|----------------|
| Simplicidad | Media: más servicios y disciplina de apagado | Media: pocos servicios pero adapter menos común | **Alta** |
| Mantenibilidad | Alta: arquitectura estándar y documentada | Media: Iceberg y Athena añaden piezas | Media: el estado local no se comparte |
| Rendimiento | **Alto**: columnar + streaming | Medio: segundos por consulta, coste por escaneo | Alto en local, sin concurrencia |
| Impacto en fases posteriores | **Ninguno**: coincide con M1–M7 | Alto: reescribir §3, §5, M1–M3 y ADR | Alto: desaparecen Kinesis y Redshift del roadmap |
| Coste en 25 USD/mes | Solo con la prueba de Redshift y Kinesis apagado | **Sí** | **Sí** |
| Valor para la entrevista (CLAUDE.md §1) | **Alto** | Alto (lakehouse es actual) | Bajo |
| RGPD / supresión | Bueno (DELETE en Redshift + reescritura S3) | **Muy bueno** (Iceberg) | Frágil |

- **Elegir P1** si la cuenta es elegible para la prueba de Redshift y se acepta destruir el entorno cuando no se usa.
- **Elegir P2** si el presupuesto es la restricción dura y se acepta escribir un ADR que cambie el warehouse.
- **Elegir P3** solo si se quiere una demo mínima y rápida.
- **Gaps comunes:** precios de eu-west-3 sin verificar; la lista de servicios del Free plan no cargó; las tres necesitan Paid plan antes de 6 meses.

**Recomendación inicial (antes de la respuesta del usuario): Proposal 1**, con Kinesis detrás de `enable_streaming` (apagado por defecto) y la prueba de Redshift activada al empezar M1. Es la única sin impacto en las fases posteriores y con el relato completo de la entrevista. El presupuesto de 25 USD se protege con el budget de M0 y con el crédito de 170 USD, que se conserva al pasar a Paid plan.

**Recomendación revisada tras seguir en Free plan: Proposal 3** (superada por la siguiente), porque solo usa servicios básicos (Lambda, API Gateway, S3), que es lo que con más probabilidad permite el Free plan. P2 solo si se confirma en la consola que Athena y Firehose están disponibles.

**Recomendación final (2026-11-06): Proposal 2b.** Cumple las tres restricciones del usuario (Free plan, 25 USD/mes y vida de semanas), solo usa servicios comprobados en la consola y conserva un warehouse en AWS con SQL, dbt, SCD2 y un borrado RGPD sólido, que P3 no tiene. P3 sigue siendo la opción más simple si prima terminar rápido.

## Decision

_Pendiente de la elección del usuario._
