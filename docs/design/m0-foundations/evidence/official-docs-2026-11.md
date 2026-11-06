# Evidencia — documentación oficial consultada

Material externo, inmutable. Citas literales leídas el 2026-11-05 y el 2026-11-06 con un subagente de lectura web.
Si una página se vuelve a consultar, se crea un fichero nuevo con su fecha; este no se edita.

## AWS Free plan
- URL: https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/free-tier-plans.html
- "Your free account plan ends after six months or when your credits are fully used - whichever occurs first."
- "After your free account plan expires, your account closes automatically, and you lose access to your resources and data."
- "Any remaining credits from your free account plan are automatically applied to future AWS bills."
- "Free account plans don't include access to AWS services and features that could possibly deplete your credits, or hardware purchases."
- La página de restricciones (https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/free-plan-service-restrictions.html) devolvió solo "AWS Billing", sin contenido, en ambas consultas.

## Kinesis Data Streams
- URL: https://aws.amazon.com/kinesis/data-streams/pricing/
- "Amazon Kinesis Data Streams is NOT currently available in AWS Free Tier."
- "One shard costs $0.015 per hour" (ejemplo US-East).

## Amazon Data Firehose
- URL: https://aws.amazon.com/firehose/pricing/
- "Ingestion pricing is tiered and billed per GB ingested in 5KB increments (a 3KB record is billed as 5KB, a 12KB record is billed as 15KB, etc.)."

## Redshift Serverless
- URL: https://aws.amazon.com/redshift/pricing/ (página truncada al leerla)
- "Redshift Serverless begins at $1.50 per hour"; "You can adjust this capacity from 4 to 1024 RPUs."
- "You pay for the workloads you run in RPU-hours on a per-second basis (with a 60-second minimum charge)."
- URL: https://aws.amazon.com/redshift/free-trial/ — "The Amazon Redshift free trial program is not part of the AWS Free Tier."

## Amazon Athena
- URL: https://aws.amazon.com/athena/pricing/
- Ejemplo: "Price for 3 TB scanned is 3 * $5/TB = $15." (el precio por TB sale del ejemplo, no de una línea de precio).

## AWS Budgets
- URL: https://aws.amazon.com/aws-cost-management/aws-budgets/pricing/
- "You can monitor and receive notifications on your budgets free of charge."
- "Your first two action-enabled budgets are free ... Afterwards each subsequent action-enabled budget will incur a $0.10 daily cost."
- Recurso Terraform: https://raw.githubusercontent.com/hashicorp/terraform-provider-aws/main/website/docs/r/budgets_budget.html.markdown — `time_unit` "Valid values: `MONTHLY`, `QUARTERLY`, `ANNUALLY`, and `DAILY`"; `threshold_type` "Can be `PERCENTAGE` OR `ABSOLUTE_VALUE`"; `notification_type` "Can be `ACTUAL` or `FORECASTED`".

## Terraform S3 backend
- URL: https://developer.hashicorp.com/terraform/language/backend/s3 (docs v1.17.x)
- "This backend also supports state locking which can be enabled by setting the `use_lockfile` argument to `true`."
- "DynamoDB-based locking is deprecated and will be removed in a future minor version."
- "If `use_lockfile` is set, `s3:GetObject`, `s3:PutObject`, and `s3:DeleteObject` are required on the lock file."

## GitHub OIDC en AWS
- URL: https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services
- "For the provider URL: Use `https://token.actions.githubusercontent.com`"; "Use `sts.amazonaws.com` if you are using the official action."
- "The job or workflow run requires a `permissions` setting with `id-token: write`"
- Repos creados después del 15-07-2026 o con la opción activada: `repo:octo-org@123456/octo-repo@456789:ref:refs/heads/octo-branch`.
- URL: https://github.com/aws-actions/configure-aws-credentials — major v6 (ejemplos `@v6.3.0`); "The thumbprint, if specified, will be ignored."

## pre-commit
- URL: https://pre-commit.com/ — "create a file named `.pre-commit-config.yaml`"; "run `pre-commit run --all-files`".
