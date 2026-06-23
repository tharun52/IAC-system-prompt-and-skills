write-terraform-code

Generate production-ready Terraform infrastructure using Terraform MCP as the source of truth. Follow Terraform best practices, use the latest provider and resource documentation, and organize code into the standard repo structure.

# Skill: Write Terraform Code

## Objective

Generate production-ready Terraform code based on the approved architecture plan.

## Workflow

### Step 1 — Read the Approved Plan

Use the architecture plan from the requirements phase as the source of truth. Do not deviate from it without user confirmation.

### Step 2 — Look Up Providers and Resources

Use the Terraform MCP registry tools before writing any code:

| Tool | When to use |
|------|-------------|
| `get_latest_provider_version` | Get the latest version of the AWS provider to pin in `versions.tf` |
| `search_providers` | Find the right provider or resource documentation by name (e.g. `aws_vpc`) |
| `get_provider_details` | Fetch the full documentation for a specific resource or data source — required and optional arguments, attribute references, and examples. Use this for every resource before writing it |
| `search_modules` | Find a public registry module if a well-known module exists for the resource (e.g. VPC, EKS) |
| `get_module_details` | Get full inputs, outputs, and usage examples for a module |
| `get_latest_module_version` | Get the latest version of a module to pin in the module block |

Do not rely on training data for resource arguments — always call `get_provider_details` before writing each resource block.

### Step 3 — Generate the Repo Structure

The repository uses this layout — every resource group gets its own subfolder inside `terraform/`:

```
repo/
├── env/
│   ├── dev.tfvars
│   ├── staging.tfvars
│   └── prod.tfvars
└── terraform/
    ├── versions.tf           # terraform {} block and S3 backend only
    ├── providers.tf          # provider {} blocks only
    ├── variables.tf
    ├── outputs.tf
    ├── main.tf               # calls each resource subfolder as a local module
    ├── terraform.tfvars      # active environment values (used by pipeline)
    ├── vpc/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    ├── compute/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── storage/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

**Each resource group must be its own subfolder** — do not put resource blocks directly in the root `terraform/main.tf`. The root `terraform/main.tf` only contains `module` blocks that call each subfolder. Name subfolders by resource group (e.g. `vpc`, `compute`, `database`, `storage`, `iam`).

Keep `versions.tf` and `providers.tf` separate — `versions.tf` contains only the `terraform {}` block, `providers.tf` contains only `provider {}` blocks.

### Step 4 — Remote State

Always configure an S3 remote state backend in `versions.tf` using the bucket name collected during requirements gathering (from the system prompt or the user's answer). Terraform does not support variables in the backend block — use literal values only:

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

All configurable values must be defined as variables in `terraform/variables.tf`. No hardcoded values anywhere in the code.

Every variable must have a `description` and `type`. Include a `default` only where a safe, sensible default exists.

Common variables to always include:
- `aws_region`
- `environment`
- `project_name`

Generate environment-specific tfvars files in `env/` (e.g., `env/dev.tfvars`, `env/prod.tfvars`) with the actual values collected during requirements gathering.

Copy the active environment's values into `terraform/terraform.tfvars` — this is the file the pipeline reads at runtime.

### Step 6 — Terraform Standards

- Pin provider versions
- Use `locals` for repeated values
- Tag all resources using `environment` and `project_name` variables
- Follow least-privilege IAM
- Document all variables and outputs

### Step 7 — Format Before Committing

The pipeline runs `terraform fmt -check -recursive` on every push. If any file fails the check, the entire pipeline fails. Apply these rules to every `.tf` and `.tfvars` file before committing:

**Indentation** — 2 spaces, no tabs:
```hcl
resource "aws_s3_bucket" "this" {
  bucket = var.bucket_name
}
```

**Aligned `=` signs** — align within each block (not globally):
```hcl
# correct
bucket      = var.bucket_name
environment = var.environment

# wrong
bucket = var.bucket_name
environment = var.environment
```

**Blank lines** — one blank line between top-level blocks, no blank line after the opening `{`:
```hcl
resource "aws_s3_bucket" "this" {
  bucket = var.bucket_name
}

resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id
}
```

**No trailing whitespace** on any line.

**No extra spaces inside block labels** — one space between the closing `"` and `{`:
```hcl
# correct
resource "aws_vpc" "main" {

# wrong
resource "aws_vpc" "main"  {   # extra space before {
```

**tfvars files** — align `=` signs:
```hcl
aws_region   = "us-east-1"
environment  = "dev"
project_name = "my-project"
```

When in doubt, mentally apply `terraform fmt` before writing each block — misaligned assignments and wrong indentation are the most common causes of fmt failures.

### Step 8 — Commit Files and Raise PR

Use the GitHub MCP to:
1. Create a feature branch named `terraform/{project_name}`
2. Use `push_files` to commit all generated files to that branch in a single commit — pass all files as an array, do not call `create_or_update_file` one file at a time
3. Raise a pull request to main

The PR body must use proper markdown with real line breaks — not escaped `\n` characters. Structure it as:

```
## Summary
Brief description of the infrastructure being created.

## Resources
| Resource | Details |
|----------|---------|
| ...      | ...     |

## Remote State
- Bucket: ...
- Key: ...

## Pipeline
Pushing to this branch triggers fmt, validate, checkov, and plan automatically.
Merging this PR will run plan and apply automatically.
For a manual run, go to GitHub Actions → Manual workflow → Run workflow and select the action.
```
