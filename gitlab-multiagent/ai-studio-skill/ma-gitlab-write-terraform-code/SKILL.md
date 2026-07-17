---
name: ma-gitlab-write-terraform-code
description: "Generate production-ready Terraform code from the approved plan provided by the orchestrator and commit it to GitLab."
---

## Workflow

### Step 1 — Read the Approved Plan

Use the plan as the source of truth. Do not deviate from it.

### Step 2 — Terraform MCP Lookups (do this before writing any code)

Make these calls **in parallel in a single round**:

1. `get_latest_provider_version` for `hashicorp/aws` — pin this in `versions.tf`. Call this **once**, not per resource.
2. `get_provider_details` for **every resource type in the plan** — call all in parallel, not one at a time. Never rely on training data for resource arguments.
3. If the plan uses a module: `get_module_details` + `get_latest_module_version` in the same parallel round.

Do **not** call `search_providers` — the provider is always `hashicorp/aws`. Do **not** call `search_modules` unless the plan explicitly names a module to find.

### Step 3 — Generate the Repo Structure

All `.tf` files go directly inside `terraform/` — never in subfolders. Do not create a project-name folder, do not create layer folders (networking/, compute/, iam/, etc.). One `.tf` file per resource group, all flat.

```
repo/
├── env/
│   └── {environment}.tfvars
└── terraform/
    ├── backend.tf
    ├── versions.tf
    ├── providers.tf
    ├── variables.tf
    ├── outputs.tf
    ├── main.tf
    ├── terraform.tfvars
    ├── {resource-a}.tf      ← e.g. s3.tf, lambda.tf, iam.tf, vpc.tf
    └── {resource-b}.tf
```

Never create `terraform/{project_name}/` or any subfolder inside `terraform/`.

### Step 4 — backend.tf

S3 backend only, literal values (no `var.*`), key format `{project_name}/{environment}/terraform.tfstate`:

```hcl
terraform {
  backend "s3" {
    bucket  = "{state_bucket}"
    key     = "{project_name}/{environment}/terraform.tfstate"
    region  = "{aws_region}"
    encrypt = true
    assume_role = {
      role_arn = "arn:aws:iam::{assume_role_account_id}:role/{assume_role_name}"
    }
  }
}
```

### Step 5 — versions.tf

`required_version` and `required_providers` only. Use the version from Step 2. Always include `random`:

```hcl
terraform {
  required_version = ">= 1.3.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> X.X"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.9"
    }
  }
}
```

### Step 6 — providers.tf

Provider blocks only. No static credentials — runner uses IAM instance role with `assume_role`:

```hcl
provider "aws" {
  region = var.region

  assume_role {
    role_arn     = "arn:aws:iam::{assume_role_account_id}:role/{assume_role_name}"
    session_name = "terraform-gitlab-runner"
  }
}

provider "random" {}
```

### Step 7 — Variablize Everything

All configurable values in `terraform/variables.tf`. Always include `aws_region`, `environment`, `project_name`. Generate `env/{environment}.tfvars` and copy into `terraform/terraform.tfvars`.

### Step 8 — Terraform Standards

- Use `locals` for repeated values
- Tag all resources with `environment` and `project_name`
- Follow least-privilege IAM
- Document all variables and outputs

### Step 9 — Write Correctly Formatted Code

The pipeline catches fmt errors and `ma-gitlab-fix-pipeline` fixes them — do not run local checks.

- 2-space indentation, no tabs
- Aligned `=` signs within each block
- One blank line between top-level blocks, no blank line after opening `{`
- No trailing whitespace, no inline comments after values

### Step 10 — Commit and Raise MR

1. Create branch `terraform/{project_name}`
2. Commit files using full relative paths (e.g. `terraform/networking/main.tf`):
   - `push_files` — new files only, batch by subfolder
   - `edit_files` — existing files only, search + replace strings, all files in one atomic commit
3. Raise MR to main with `create_merge_request` (real line breaks, not `\n`)

Return the MR link to the orchestrator.
