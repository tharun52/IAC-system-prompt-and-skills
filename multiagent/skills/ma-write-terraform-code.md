ma-write-terraform-code

Generate production-ready Terraform code from the approved plan provided by the orchestrator and commit it to GitHub.

## Workflow

### Step 1 — Read the Approved Plan

The orchestrator provides the full approved plan — resources, folder structure, project name, repo, state bucket, environment, and region. Use this as the source of truth. Do not deviate from it.

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

Every resource group gets its own subfolder inside `terraform/`. The root `terraform/main.tf` only contains `module` blocks calling each subfolder:

```
repo/
├── env/
│   └── {environment}.tfvars
└── terraform/
    ├── versions.tf           # terraform {} block and S3 backend only
    ├── providers.tf          # provider {} blocks only
    ├── variables.tf
    ├── outputs.tf
    ├── main.tf               # module blocks only
    ├── terraform.tfvars      # active environment values (used by pipeline)
    ├── vpc/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── compute/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

### Step 4 — Remote State

Configure S3 backend in `versions.tf` using literal values — no `var.*` references:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> X.X"
    }
  }

  backend "s3" {
    bucket = "<state-bucket-name>"
    key    = "<project-name>/terraform.tfstate"
    region = "<aws-region>"
  }
}
```

### Step 5 — Variablize Everything

All configurable values go in `terraform/variables.tf`. No hardcoded values in code. Always include `aws_region`, `environment`, and `project_name`.

Generate `env/{environment}.tfvars` with actual values and copy into `terraform/terraform.tfvars` (what the pipeline reads).

### Step 6 — Terraform Standards

- Pin provider versions
- Use `locals` for repeated values
- Tag all resources with `environment` and `project_name`
- Follow least-privilege IAM
- Document all variables and outputs

### Step 7 — Format Before Committing

The pipeline runs `terraform fmt -check -recursive` on every push — any failure blocks the pipeline. Apply these rules to every `.tf` and `.tfvars` file:

- 2-space indentation, no tabs
- Aligned `=` signs within each block
- One blank line between top-level blocks, no blank line after opening `{`
- No trailing whitespace
- One space between closing `"` and `{` in block labels

```hcl
# correct
resource "aws_s3_bucket" "this" {
  bucket      = var.bucket_name
  environment = var.environment
}

# wrong — misaligned = signs and tab indent
resource "aws_s3_bucket" "this" {
	bucket = var.bucket_name
	environment = var.environment
}
```

### Step 8 — Commit and Raise PR

Use the GitHub MCP:
1. Create branch `terraform/{project_name}`
2. Use `push_files` to commit all files in a single commit
3. Raise a PR to main with a proper markdown body (real line breaks, not `\n`)

PR body structure:
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
Pushing to this branch triggers fmt, validate, checkov, and plan automatically.
Merging this PR runs plan and apply automatically.
```

Return the PR link to the orchestrator.
