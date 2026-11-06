---
type: Plan
---

# m0-foundations: Cimientos del repo — Design Document

## Current Context

- No existe ninguno de los cinco componentes de M0: no hay `Makefile`, `pyproject.toml`, `.pre-commit-config.yaml`, `.github/` ni `infra/` (research §7–8).
- El repo es público, `main` está protegida (PR obligatorio, historia lineal) y no exige status checks (research §10). El `sub` de OIDC usa IDs inmutables: `repo:javiladino@97758532/trueflow@1447219459` (findings "GitHub OIDC").
- La cuenta AWS `437266392155` está en Free plan, y en local no hay AWS CLI ni credenciales (research §8). El usuario usará IAM Identity Center.
- Arquitectura elegida: ruling "Arquitectura lakehouse (P2b)" (CONTEXT § Conventions). En M0 solo se crea la base: estado remoto, OIDC y budget, sin recursos de datos (ruling "Arquitectura" del topic).
- Versiones estables consultadas el 2026-11-05 (research §15).

## Requirements

### Functional Requirements
- **Tooling:** `make setup` deja listo el entorno con Python 3.12 vía uv, las herramientas de desarrollo con versión fijada y los hooks de pre-commit instalados.
- **pre-commit:** `pre-commit run --all-files` ejecuta los mismos checks que la CI: ruff (lint + format), mypy, terraform fmt/validate, tflint, checkov, actionlint, detección de secretos y Conventional Commits en el mensaje de commit.
- **CI:** en cada PR, un workflow ejecuta `lint` (pre-commit) y `terraform-plan`. Este último usa OIDC y publica el plan de `envs/dev` como comentario en el PR.
- **Backend de Terraform:** un stack `bootstrap` crea el bucket de estado con versionado, cifrado, bloqueo de acceso público y política solo-TLS. `envs/dev` usa el backend S3 con `use_lockfile = true`.
- **OIDC:** el provider `token.actions.githubusercontent.com` y un rol `trueflow-ci-plan` de solo lectura, más acceso al estado y al lock, con una trust policy restringida al `sub` inmutable del repo.
- **Presupuesto:** `aws_budgets_budget` COST mensual de 25 USD con alertas por email al 50 % real, al 80 % real y al 100 % previsto. El email llega por variable y nunca está en el repo.
- **Protección:** cuando la CI esté en verde, `lint` y `terraform-plan` pasan a ser checks obligatorios en `main`.

### Non-Functional Requirements
- **Coste:** M0 no crea nada con coste fijo. S3 de estado es menos de 1 MB y Budgets es gratis (findings "Coste").
- **Seguridad:** sin claves estáticas (OIDC + Identity Center); mínimo privilegio en el rol de CI; nada sensible en el repo, porque es público.
- **Reproducibilidad:** todas las versiones fijadas en ficheros versionados.

## Design Decisions

### 1. Autenticación local: IAM Identity Center
Respuesta del usuario (grill-me, 2026-11-06). Motivos: credenciales temporales y sin claves en disco. Alternativa descartada: un usuario IAM con access key de larga duración.

### 2. Bootstrap del estado: stack aparte con estado local y migración
El backend necesita el bucket antes del primer `init` (research §14). `infra/terraform/bootstrap` se aplica con estado local y después se migra a su propio bucket con `terraform init -migrate-state`. Alternativa descartada: crear el bucket a mano, que deja recursos sin código.

### 3. Locking con `use_lockfile`, sin DynamoDB
DynamoDB está deprecado (findings "Terraform"). Además evita una tabla extra.

### 4. La CI de M0 solo hace plan, sin apply
El rol `trueflow-ci-plan` es de solo lectura. El despliegue en `main` (CLAUDE.md §5) queda para M1, cuando haya recursos que desplegar; así un fallo en M0 no puede crear coste.

### 5. Herramientas de Python con uv, el resto con brew
`pyproject.toml` en la raíz con un grupo `dev` (ruff, mypy, pytest, pre-commit, sqlfluff) fijado con `uv.lock`. Terraform 1.17.1 (desde el tap `hashicorp/tap`), tflint y AWS CLI se instalan con brew. checkov y actionlint corren como hooks de pre-commit en entornos aislados. Go y golangci-lint se instalan en M2, cuando exista `collector/`.

### 6. dbt fuera de M0
Sin proyecto dbt no hay nada que construir. El paso `dbt build` en `pr_<n>` de CLAUDE.md §5 se añade en M1.

## Technical Design

### 1. Core Components
- `Makefile`: objetivos `setup`, `lint`, `test`, `tf-plan ENV=dev`. Los de §11 que dependen de fases posteriores imprimen "not implemented until M<n>" y salen con código 0.
- `infra/terraform/bootstrap`: bucket `trueflow-tfstate-437266392155-euw3`.
- `infra/terraform/modules/github-oidc`: entradas `sub_prefix`, `state_bucket` y `role_name`; salida `role_arn`.
- `infra/terraform/modules/budget`: entradas `limit_usd = 25` y `alert_emails` (lista, sensible); sin salidas.
- `infra/terraform/envs/dev`: compone los dos módulos con `default_tags` (`project`, `env`, `owner`, `cost-center`).
- `.github/workflows/ci.yml`: jobs `lint` y `terraform-plan`, con `permissions: id-token: write, contents: read, pull-requests: write`.

### 2. Data Models
- `terraform.tfvars.example`: `budget_alert_emails = ["you@example.com"]`. El fichero real `terraform.tfvars` es local e ignorado.
- Secreto de GitHub `TF_VAR_budget_alert_emails`, con el valor en formato JSON, para el plan en la CI.

### 3. Integration Points
- GitHub OIDC → STS `AssumeRoleWithWebIdentity` → `trueflow-ci-plan` (condiciones: `aud = sts.amazonaws.com`, `sub` con `StringLike` sobre `repo:javiladino@97758532/trueflow@1447219459:*`).
- `aws-actions/configure-aws-credentials@v6` con `aws-region: eu-west-3`.

### 4. Files Changed
- `.gitignore` (lines 27–31: añadir `*.tfvars` y `!*.tfvars.example` en el bloque de Terraform)
- `CLAUDE.md` (line 101: Bedrock disponible en el Free plan, comprobado por el usuario)
- `docs/architecture/architecture.md` (line 40: quitar "por verificar" de Bedrock)
- `docs/design/findings.md` (lines 7–11: añadir Bedrock al finding del Free plan)
- `docs/design/CONTEXT.md` (line 30: quitar Bedrock de Known issues)
- `.python-version` (NEW: `3.12`)
- `pyproject.toml` (NEW: proyecto `trueflow`, `requires-python = ">=3.12,<3.13"`, grupo `dev` con ruff 0.17.3, mypy 2.4.1, pytest 9.2.0, pre-commit 4.7.0, sqlfluff 4.5.0; configuración de ruff y mypy)
- `uv.lock` (NEW: generado por `uv lock`)
- `.pre-commit-config.yaml` (NEW: hooks de pre-commit-hooks, ruff, mypy, terraform fmt/validate/tflint, checkov 3.4.3, actionlint, gitleaks y conventional-pre-commit en `commit-msg`)
- `.tflint.hcl` (NEW: plugin `aws`, regla de tags obligatorios)
- `Makefile` (NEW: objetivos de CLAUDE.md §11)
- `.env.example` (NEW: `AWS_PROFILE=trueflow-dev`, `AWS_REGION=eu-west-3`)
- `infra/terraform/bootstrap/{versions,main,variables,outputs,backend}.tf` (NEW: bucket de estado; `backend.tf` se añade después de la migración)
- `infra/terraform/modules/github-oidc/{main,variables,outputs,versions}.tf` (NEW)
- `infra/terraform/modules/budget/{main,variables,versions}.tf` (NEW)
- `infra/terraform/envs/dev/{versions,backend,providers,main,variables}.tf`, `terraform.tfvars.example` (NEW; `required_version = "~> 1.17.0"`, provider `aws ~> 6.72`)
- `.github/workflows/ci.yml` (NEW)
- `docs/cost.md` (NEW: estimación por componente para M0 y la arquitectura P2b)
- `docs/runbooks/budget-alert.md` (NEW: qué hacer cuando llega una alerta del budget)
- `docs/interview/m0-foundations.md` (NEW, en la fase T)

## Implementation Plan

1. **Fase 1 — Correcciones de documentación y repo** (sin AWS)
   - Bedrock confirmado en CLAUDE.md, el diagrama, findings y CONTEXT; `.gitignore` con `*.tfvars`.
   - Validación: `git check-ignore infra/terraform/envs/dev/terraform.tfvars` imprime la ruta.
2. **Fase 2 — Tooling de Python y pre-commit** (sin AWS)
   - `.python-version`, `pyproject.toml`, `uv.lock`, `.pre-commit-config.yaml`, `Makefile`, `.env.example`.
   - Validación: `make lint` sale con 0.
3. **Fase 3 — Código de Terraform** (sin AWS: solo `init -backend=false` y `validate`)
   - `bootstrap`, los módulos `github-oidc` y `budget`, `envs/dev`, `.tflint.hcl`.
   - Validación: `make lint` sale con 0, con terraform validate, tflint y checkov incluidos.
4. **Fase 4 — Acceso del usuario a AWS** (manual, la hace el usuario; 🚦)
   - Activar IAM Identity Center, crear un usuario con AdministratorAccess, instalar AWS CLI y ejecutar `aws configure sso` con el perfil `trueflow-dev`.
   - Validación: `aws sts get-caller-identity --profile trueflow-dev` devuelve la cuenta `437266392155`.
5. **Fase 5 — Bootstrap y apply de dev** (🚦: confirmación explícita del usuario antes de cada `apply`)
   - `terraform apply` en `bootstrap`, migración de su estado al bucket, y `terraform apply` en `envs/dev` (OIDC, rol y budget).
   - Validación: `aws budgets describe-budgets --account-id 437266392155 --profile trueflow-dev` muestra `trueflow-monthly` con un límite de 25 USD.
6. **Fase 6 — CI y protección de `main`**
   - `ci.yml`; secreto `TF_VAR_budget_alert_emails` (lo crea el usuario con `gh secret set`); PR de la rama; añadir los checks `lint` y `terraform-plan` como obligatorios en `main`.
   - Validación: `gh pr checks` muestra `lint` y `terraform-plan` en pass, y el PR tiene un comentario con el plan.

## Testing Strategy

M0 no tiene código de aplicación: no hay tests unitarios que escribir. `make test` imprime "no tests until M1" y sale con 0. Estos son los checks que ejecutará `reppit-test`:

| # | Check | Comando | Resultado esperado |
|---|---|---|---|
| 1 | Lint completo | `make lint` | exit 0 |
| 2 | Hooks iguales que la CI | `pre-commit run --all-files` | todos `Passed` |
| 3 | Sin secretos ni email en el repo | `git grep -n "@gmail.com"` | sin resultados |
| 4 | tfvars ignorado | `git check-ignore infra/terraform/envs/dev/terraform.tfvars` | imprime la ruta |
| 5 | Locking nativo | `git grep -n "use_lockfile = true" infra/terraform` | aparece en `envs/dev/backend.tf` y `bootstrap/backend.tf` |
| 6 | Sin DynamoDB | `git grep -n "dynamodb" infra/terraform` | sin resultados |
| 7 | Bucket de estado endurecido | `aws s3api get-bucket-versioning --bucket trueflow-tfstate-437266392155-euw3 --profile trueflow-dev` | `"Status": "Enabled"` |
| 8 | Budget | `aws budgets describe-budgets --account-id 437266392155 --profile trueflow-dev` | límite 25 USD, `MONTHLY`, 3 notificaciones |
| 9 | Trust policy con `sub` inmutable | `aws iam get-role --role-name trueflow-ci-plan --profile trueflow-dev` | condición `StringLike` con `repo:javiladino@97758532/trueflow@1447219459:*` |
| 10 | CI con OIDC | `gh pr checks` | `lint` y `terraform-plan` pass |
| 11 | Checks obligatorios | `gh api repos/javiladino/trueflow/branches/main/protection/required_status_checks` | `contexts` contiene `lint` y `terraform-plan` |
| 12 | Tags obligatorios | `make lint` (regla de tflint) | ningún recurso sin los 4 tags |
| 13 | Plan limpio tras el apply | `make tf-plan ENV=dev` | "No changes." |
| 14 | Coste | Cost Explorer al final de M0 | < 1 USD |

## Observability

Not applicable for this change. Los monitores son de M5; el budget es la única alerta de M0 y tiene runbook.

## Future Considerations

### Known Limitations
- No hay rol de despliegue para `main` hasta M1 (Design Decision 4).
- Go, golangci-lint y el paso de dbt en la CI se añaden en M2 y M1.
- Pendiente para M1: soporte de `dbt snapshot` y unit tests en `dbt-athena`.

## Dependencies

### Development Dependencies
Versiones de research §15: Terraform 1.17.1, provider AWS 6.72.0, tflint 0.65.0, checkov 3.4.3, pre-commit 4.7.0, ruff 0.17.3, mypy 2.4.1, pytest 9.2.0, sqlfluff 4.5.0, `aws-actions/configure-aws-credentials@v6`. AWS CLI v2 con brew.

## Security Considerations
- Rol de CI: `ReadOnlyAccess` más `s3:GetObject`/`PutObject`/`DeleteObject` sobre `*.tflock` y `s3:GetObject`/`ListBucket` sobre el estado. Ningún permiso de escritura en la cuenta.
- Trust policy limitada al `sub` inmutable del repo y a `aud = sts.amazonaws.com`.
- Bucket de estado: versionado, SSE-S3, `BlockPublicAccess` completo y política que deniega peticiones sin TLS.
- El email de las alertas vive solo en `terraform.tfvars` local (ignorado) y en un secreto de GitHub.
- gitleaks en pre-commit; checkov sobre Terraform.

## Rollout Strategy
1. Fases 1–3 en `feat/001-m0-foundations`, sin tocar AWS.
2. Fase 4 manual (usuario).
3. Fase 5 con confirmación explícita por cada `apply`. Rollback: `terraform destroy` en `envs/dev`, y después en `bootstrap` tras vaciar el bucket. Ninguno de los dos tiene datos de negocio.
4. Fase 6: merge del PR con checks en verde.

## References
- `docs/design/m0-foundations/working/research.md` (§7–15)
- `docs/design/m0-foundations/working/proposal.md` (P2b, § Decision)
- `docs/adr/0001-lakehouse-on-free-plan.md`
- `docs/design/findings.md`
- Docs oficiales citados en el research: Terraform S3 backend, GitHub OIDC en AWS, configure-aws-credentials, `aws_budgets_budget`, pre-commit.
