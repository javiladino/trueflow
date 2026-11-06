# ADR 0001 — Lakehouse en S3 con Athena e Iceberg, sin streaming

- **Estado:** Aceptado · **Fecha:** 2026-11-06 · **Decide:** Javier (owner)
- **Discusión:** `docs/design/m0-foundations/working/proposal.md` (Proposal 2b y § Decision)

## Contexto

La arquitectura inicial usaba Kinesis Data Streams + Firehose y Redshift Serverless. Restricciones reales:

- Cuenta AWS en **Free plan** con 170 USD de crédito; el proyecto es educativo, vive unas semanas y se desmantela.
- Presupuesto de **25 USD/mes**.
- Kinesis Data Streams "is NOT currently available in AWS Free Tier"; la prueba de Redshift Serverless
  "is not part of the AWS Free Tier"; Firehose exige pasar a Paid plan (comprobado por el owner en la consola).
- Athena y Glue sí abren en el Free plan (comprobado por el owner en la consola).

## Decisión

- El collector (Go) escribe los eventos agrupados directamente en S3 raw (`events/dt=/hour=/`).
- Warehouse: **Athena + Glue Data Catalog + tablas Iceberg** en S3; dbt con **`dbt-athena`**.
- El forwarder a Amplitude se dispara con notificaciones `s3:ObjectCreated`.
- Se eliminan Kinesis, Firehose y Redshift de la arquitectura.

## Consecuencias

- **+** Cabe en el Free plan y en el presupuesto (Athena: ejemplo oficial de 5 USD/TB escaneado).
- **+** Iceberg permite `DELETE`/`MERGE` por fila: el derecho de supresión es más sencillo.
- **+** Desmantelamiento simple: `terraform destroy` y vaciar buckets.
- **−** Sin buffer de streaming: muchos objetos pequeños → agrupar en el collector y compactar Iceberg.
- **−** Sin fan-out ni orden por shard; el relato de entrevista pasa de "streaming + warehouse" a "lakehouse".
- **−** Por verificar al inicio del plan: soporte de `dbt snapshot` (SCD2) y de unit tests en `dbt-athena`.
- **−** Athena cobra por bytes escaneados: Metabase debe consultar marts particionados y pequeños.

## Qué haría a mayor escala

Volver a un buffer de streaming (Kinesis o MSK) delante de S3 y valorar un warehouse con cómputo dedicado
(Redshift) cuando la concurrencia de dashboards o la latencia lo justifiquen.
