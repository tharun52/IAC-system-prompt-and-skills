---
name: ma-gitlab-write-terraform-code
description: "Generate production-ready Terraform code from the approved plan provided by the orchestrator and commit it to GitLab."
---

## Workflow

### Step 1 — Read the Approved Plan

The orchestrator provides the full approved plan — resources, folder structure, project name, state bucket, environment, and region. Use this as the source of truth. Do not deviate from it.

### Step 2 — Look Up Providers and Resources

Use the Terraform MCP before writing any code:

| Tool | When to use |
|------|-------------|
| `get_latest_provider_version` | Get the latest AWS provider version to pin in `versions.tf` |
| `search_providers` | Find provider documentation by resource name |
| `get_provider_details` | Fetch required and optional arguments for a resource — call this before writing every resource block |
| `search_modules` | Find a public module for the resource (e.g. VPC, EKS) |
| `get_module_details` | Get inputs, outputs, and examples for a module |
| `get_latest_module_version` | Get the latest version of a module |

### Step 3 — Generate the Repo Structure

Always use subfolders — never put resource `.tf` files flat in `terraform/`. Group resources by logical layer. The root `terraform/main.tf` only contains `module` blocks calling each subfolder.

Typical groupings:
- `networking/` — VPC, subnets, route tables, security groups
- `compute/` — EC2, launch templates, ASG
- `storage/` — S3, EFS
- `iam/` — roles, policies, instance profiles
- `alb/` — load balancer, target groups, listeners

```
repo/
├── env/
│   └── {environment}.tfvars
└── terraform/
    ├── backend.tf            # S3 backend block only
    ├── versions.tf           # required_version and required_providers only
    ├── providers.tf          # provider {} blocks only
    ├── variables.tf
    ├── outputs.tf
    ├── main.tf               # module blocks only
    ├── terraform.tfvars      # active environment values (used by pipeline)
    ├── networking/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    ├── compute/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── iam/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

If the validation agent returned a flat file list (e.g. `s3.tf`, `asg.tf`, `iam.tf` all in `terraform/`), re-map them into the subfolder structure above before writing any code.

### Step 4 — backend.tf

Create a separate `backend.tf` — do not put the backend block in `versions.tf`. Use literal values only — no `var.*` references. The key format is `{project_name}/{environment}/terraform.tfstate`. Use the state bucket and assume role details from your system prompt:

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

`versions.tf` contains only `required_version` and `required_providers`. Use the latest AWS provider version from `get_latest_provider_version`. Always include the `random` provider:

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

`providers.tf` contains provider blocks only. No static credentials or environment variables — the runner's IAM instance role assumes the cross-account role. Use the account ID and role name from your system prompt:

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

All configurable values go in `terraform/variables.tf`. No hardcoded values in code. Always include `aws_region`, `environment`, and `project_name`.

Generate `env/{environment}.tfvars` with actual values and copy into `terraform/terraform.tfvars` (what the pipeline reads).

### Step 8 — Terraform Standards

- Pin provider versions
- Use `locals` for repeated values
- Tag all resources with `environment` and `project_name`
- Follow least-privilege IAM
- Document all variables and outputs

### Step 9 — Write Correctly Formatted Code

Write every `.tf` and `.tfvars` file with correct formatting from the start. The pipeline will catch any fmt errors and `ma-gitlab-fix-pipeline` will fix them — do not attempt local checks or code execution.

Rules to follow:
- 2-space indentation, no tabs
- Aligned `=` signs within each block
- One blank line between top-level blocks, no blank line after opening `{`
- No trailing whitespace
- No inline comments after values

### Step 10 — Commit and Raise MR

Use the GitLab MCP:
1. Create branch `terraform/{project_name}`
2. Commit files — always pass the full relative path including subfolder (e.g. `terraform/networking/main.tf`). GitLab creates subfolders automatically from the path. Do not make separate calls to create directories.

   **Which tool to use:**
   - `push_files` — new files only (branch was just created, files do not exist yet). Split into batches per subfolder group to reduce payload: one call for root `terraform/` files, one for `networking/`, one for `compute/`, etc.
   - `edit_files` — existing files only (branch already has files). Pass only the search + replace strings for each file — never the full file content. All files are committed atomically in one call, no SHA management needed.

3. Raise an MR to main using `create_merge_request` with a proper markdown description (real line breaks, not `\n`)

MR description structure:
```
## Summary
Brief description.

## Resources
| Resource | Details |
|----------|---------|
| ...      | ...     |

## Remote State
- Bucket: ...
- Key: ...

## Pipeline
Opening this MR triggers: fmt → init → validate + checkov (parallel) → plan.
Merging this MR triggers: plan + apply automatically.
```

Return the MR link to the orchestrator.
