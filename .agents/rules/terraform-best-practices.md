---
trigger: model_decision
description: Apply to all Terraform tasks (writing, reviewing, or generating .tf/.tfvars). Covers modules, naming, state, variables, security, and lifecycle. Use whenever creating resources, modules, or infra to ensure best practices and avoid anti-patterns.
---

activation: glob
glob: "**/*.tf,**/*.tfvars,**/*.tfvars.json,**/terragrunt.hcl"
---

# Terraform — Best Practices

Read fully before writing or reviewing any Terraform configuration.
Rules are non-negotiable unless the user explicitly overrides a specific item.

> **Baseline:** Terraform ≥ 1.5 · AWS Provider ≥ 5.x · OpenTofu is a drop-in alternative.
> BUSL licensing applies to Terraform ≥ 1.6 — flag when relevant.

---

## Anti-Patterns ❌

### State & Backend

- **Never use local state for shared infra** — causes collaboration races and no locking.
  Use S3 + DynamoDB from day one.
  ```hcl
  # ✅ Terraform ≥ 1.11 — S3 native locking (DynamoDB no longer needed)
  terraform {
    backend "s3" {
      bucket       = "org-tfstate"
      key          = "services/api/terraform.tfstate"
      region       = "us-east-1"
      encrypt      = true
      use_lockfile = true   # replaces dynamodb_table; GA in 1.11, dynamodb_table deprecated
    }
  }

  # ✅ Terraform < 1.11 — legacy DynamoDB locking (still works, being deprecated)
  terraform {
    backend "s3" {
      bucket         = "org-tfstate"
      key            = "services/api/terraform.tfstate"
      region         = "us-east-1"
      encrypt        = true
      dynamodb_table = "terraform-state-lock"
    }
  }
  ```
- **Never commit `.tfstate` or `.tfstate.backup`** — state contains plaintext secrets.
- **Never run manual `terraform state` commands in CI** — causes drift and lock contention.
- **Never use `terraform_remote_state` to share data between root modules** if avoidable —
  exposes entire state. Prefer SSM Parameter Store or Consul for cross-state values.

### Module & Code Design

- **Never put everything in `main.tf` beyond small projects** — split logically.
- **Never repeat the resource type in the resource name:**
  ```hcl
  # ❌
  resource "aws_security_group" "main_security_group" {}
  # ✅
  resource "aws_security_group" "main" {}
  ```
- **Never hardcode environment-specific values** (account IDs, ARNs, IPs) — use data sources:
  ```hcl
  data "aws_caller_identity" "current" {}
  locals { account_id = data.aws_caller_identity.current.account_id }
  ```
- **Never use `-` in resource/variable/output names** — use `_`. Dashes belong in argument
  *values* only (DNS names, Name tags).
- **Never use `count` to differentiate multiple distinct instances** — use `for_each` with a map
  or set for stable identity keys.
- **Never nest more than one ternary in a single expression** — extract into `locals`.
- **Never use `terraform.tfvars` inside modules** — only at root/composition level.

### Secrets & Security

- **Never store secrets in `.tf`, `.tfvars`, or state** — use Secrets Manager, SSM, or Vault.
- **Never hardcode credentials in provider blocks:**
  ```hcl
  # ❌ access_key / secret_key in provider block
  # ✅ use assume_role, instance profile, IRSA, or env vars
  provider "aws" {
    region = var.aws_region
    assume_role { role_arn = var.deploy_role_arn }
  }
  ```
- **Never use unconstrained version ranges** (`>= 0.0.0`) for providers or modules.

### Lifecycle & Destroy Safety

- **Never omit `prevent_destroy` on stateful resources** — RDS, S3 buckets, EKS, ECR.
- **Never use `force_destroy = true` on production S3/ECR** without a justified comment.
- **Never use provisioners (`remote-exec`, `local-exec`) as first resort** — state is untracked.
  Use `user_data`, SSM documents, or Ansible instead.

---

## Best Practices ✅

### File Structure

```
module/
├── main.tf        # resources and module calls
├── variables.tf   # all input variables
├── outputs.tf     # all outputs
├── versions.tf    # required_providers + terraform version
├── locals.tf      # local values (when complex)
├── data.tf        # data sources (when > 3-4 sources)
└── README.md      # terraform-docs auto-generated

environments/
├── dev/  staging/  prod/     # each: main.tf, variables.tf, outputs.tf, versions.tf, terraform.tfvars
modules/
├── vpc/  eks/  rds/          # separate by blast radius boundary
```

### Naming Conventions

| Object | Convention | Example |
|---|---|---|
| Resource names | `snake_case`, singular, no type repeat | `aws_route_table.public` |
| Variable names | `snake_case`, units for numerics | `disk_size_gb`, `timeout_seconds` |
| Output names | `{name}_{type}_{attribute}` | `eks_cluster_endpoint` |
| Cloud resource values | `kebab-case` in Name tag/arg | `"api-service-prod"` |
| Boolean variables | Positive form | `enable_encryption` not `disable_*` |
| List/map variables | Plural noun | `private_subnet_ids` |

Use `this` when a module creates exactly one resource of that type.

### Provider & Version Pinning

```hcl
# versions.tf
terraform {
  required_version = "~> 1.9"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.80"
    }
  }
}
```
- `~>` on minor version for providers; `~>` on minor for Terraform binary.
- Commit `.terraform.lock.hcl` — never gitignore it.
- Use `tfenv` or `mise` to manage local binary versions.

### Variables

```hcl
variable "instance_type" {
  description = "EC2 instance type for API nodes."  # always required
  type        = string                               # always declare type
  default     = "t3.medium"                         # omit for env-specific values

  validation {
    condition     = contains(["t3.medium", "t3.large"], var.instance_type)
    error_message = "Must be t3.medium or t3.large."
  }
}
```
- Key order: `description` → `type` → `default` → `validation` → `nullable`.
- `nullable = false` when null must fall back to default, not propagate.
- Environment-specific vars (project_id, region): **no default** — force explicit.
- Avoid over-parameterizing: use `locals` for values with no foreseeable variation.

### Outputs

```hcl
output "eks_cluster_endpoint" {
  description = "API server endpoint for the EKS cluster."
  value       = module.eks.cluster_endpoint  # always reference resource attr, not input var
}
```
- Format: `{name}_{type}_{attribute}`. Plural name for list outputs.
- Always `description`. Use `try()` over `element(concat(...))` for conditionals.
- Never pass input variables directly as outputs — breaks implicit dependency graph.

### Locals

```hcl
locals {
  name_prefix = "${var.project}-${var.environment}"
  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "terraform"
    Owner       = var.owner_team
  }
}
```
Use locals to DRY repeated values, break complex expressions, and eliminate magic strings.

### Tagging

```hcl
provider "aws" {
  region = var.aws_region
  default_tags {
    tags = {
      Project     = var.project
      Environment = var.environment
      ManagedBy   = "terraform"
      CostCenter  = var.cost_center
    }
  }
}
```
- Use `default_tags` on the AWS provider — single source of truth.
- `tags` block in resources: last real argument, before `depends_on` / `lifecycle`.

### count vs for_each

| Use `count` for | Use `for_each` for |
|---|---|
| Binary conditional: `count = var.enable_x ? 1 : 0` | Multiple instances with stable keys |
| Simple boolean flags | Maps/sets (e.g., AZ per subnet) |

Prefer explicit boolean conditions over `length()` checks in `count` — `length()` on a
dynamic list is unknown at plan time.

### Lifecycle

```hcl
lifecycle {
  prevent_destroy       = true   # always on stateful resources
  create_before_destroy = true   # for zero-downtime replacement
  ignore_changes        = [engine_version]  # document WHY above with a comment
}
```
Never blanket `ignore_changes = all` — creates undetectable drift.

### Module Design

- Single-purpose modules: VPC module manages VPC only.
- Every module has a `README.md` (auto-generated by `terraform-docs`).
- Versioned sources only — never floating refs:
  ```hcl
  # ✅
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.16"
  # ❌
  source = "git::https://github.com/org/modules.git//vpc"  # no version pin
  ```
- For internal modules: use git tags or a private registry — never `?ref=main`.

### Secrets Management

```hcl
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "prod/myapp/db-password"
}
resource "aws_db_instance" "main" {
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
}
```
- Mark sensitive input variables with `sensitive = true`.
- Inject secrets in CI via `TF_VAR_` env vars, masked in pipeline logs.
- Use SSM `SecureString` not `String` for secrets in Parameter Store.

### State Management

```hcl
backend "s3" {
  bucket       = "org-terraform-state-prod"
  key          = "services/api/terraform.tfstate"
  region       = "us-east-1"
  encrypt      = true
  kms_key_id   = "arn:aws:kms:..."
  use_lockfile = true   # GA in Terraform 1.11; replaces dynamodb_table (deprecated)
                        # S3 bucket versioning must be enabled
                        # IAM: requires s3:DeleteObject for .tflock cleanup
}
# Migration from DynamoDB: set both use_lockfile=true AND dynamodb_table in parallel,
# verify locking works, then drop dynamodb_table.
```
- S3 bucket versioning must be enabled — required for `use_lockfile`, also serves as state DR.
- Enable S3 access logging for audit trail.
- Separate state per environment (dev/staging/prod) and per component (vpc, eks, apps).
- Never apply without a reviewed plan. Use `-target` as break-glass only — creates drift.

### Tooling & Code Style

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.99.4
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_docs
        args: ["--args=--output-file README.md"]
      - id: terraform_tflint
      - id: terraform_trivy  # or checkov
```
- `terraform fmt -check` in CI — fail on unformatted code.
- `tflint` — catches provider-specific errors `validate` misses.
- `checkov` / `trivy` — fail CI on HIGH/CRITICAL security findings.
- Comments: `#` only. Never `//` or `/* */`.
- `.editorconfig`: 2-space indent, trim trailing whitespace for `*.tf` and `*.tfvars`.

### CI/CD Pipeline Gates (in order)

1. `terraform fmt -check`
2. `terraform init -backend=false` + `terraform validate`
3. `tflint --recursive`
4. `checkov -d . --quiet --compact`
5. `terraform plan -out=tfplan` (PR gate, human review)
6. Approval gate
7. `terraform apply tfplan` (apply from saved plan only — never re-plan on apply)

Use IRSA or IAM instance profiles in CI — never static AWS keys.
Store `tfplan` as a pipeline artifact. Apply only from the saved binary plan.

### AWS-Specific Rules

- `aws_s3_bucket_public_access_block` — always block all public access.
- RDS: `deletion_protection = true`, `skip_final_snapshot = false` in production.
- IAM policies: use `aws_iam_policy_document` data source — never heredoc JSON.
- EKS: use IRSA over node IAM roles for pod-level permissions.
- Use `aws_availability_zones`, `aws_region`, `aws_caller_identity` data sources
  instead of hardcoding.

### Orchestration

- For multi-account or complex dependency graphs: use **Terragrunt** over in-house scripts.
  Terragrunt handles DRY backend config, `dependency` blocks for ordering, and env overrides.
- Do not use Terraform workspaces for environment separation in production —
  use separate directories with separate state keys. Workspaces share backend config and
  are harder to audit and gate with RBAC.