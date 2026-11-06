# trueflow — Findings

Durable facts that later topics need. Append one block per finding. Loaded only when a task touches a subject listed here. Split into `findings/{subject}.md` once this file passes about 150 lines.

## AWS Free plan: qué servicios se pueden usar

- **Date:** 2026-11-06 · **Source topic:** `m0-foundations`
- **Finding:** Cuenta en Free plan (cierra a los 6 meses o al agotar el crédito). Kinesis Data Streams no está en el Free Tier; la prueba de Redshift Serverless no forma parte del Free Tier; Firehose exige Paid plan (consola). Athena y Glue sí abren (consola, 2026-11-06).
- **Provenance:** `m0-foundations/evidence/official-docs-2026-11.md`, `m0-foundations/evidence/owner-console-checks-2026-11-06.md`

## GitHub OIDC: el `sub` del repo usa IDs inmutables

- **Date:** 2026-11-05 · **Source topic:** `m0-foundations`
- **Finding:** `use_immutable_subject: true`; el `sub` empieza por `repo:javiladino@97758532/trueflow@1447219459`. Una trust policy de IAM con `repo:javiladino/trueflow:*` no casa.
- **Provenance:** `m0-foundations/working/research.md` §10 (salida de `gh api`); `m0-foundations/evidence/official-docs-2026-11.md` § GitHub OIDC

## Terraform: locking del estado en S3

- **Date:** 2026-11-05 · **Source topic:** `m0-foundations`
- **Finding:** `use_lockfile = true` da locking nativo en S3; el locking con DynamoDB está deprecado. IAM: Get/Put/DeleteObject sobre `<key>.tflock`.
- **Provenance:** `m0-foundations/evidence/official-docs-2026-11.md` § Terraform S3 backend

## Coste: Budgets y Athena

- **Date:** 2026-11-06 · **Source topic:** `m0-foundations`
- **Finding:** Las alertas de AWS Budgets por email son gratis. Athena cobra por bytes escaneados (ejemplo oficial 5 USD/TB); precios de eu-west-3 sin verificar.
- **Provenance:** `m0-foundations/evidence/official-docs-2026-11.md` § AWS Budgets, § Amazon Athena
