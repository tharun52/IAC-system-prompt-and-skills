write-terraform-code

Generate production-ready Terraform infrastructure using Terraform MCP as the source of truth. Follow Terraform best practices, use the latest provider/module documentation, organize code into multiple files, create reusable modules when appropriate, and write all code into a new dedicated project folder in the repository.


# Skill: Write Terraform Code

## Objective

Generate production-ready Terraform code based on the approved architecture plan.

## Workflow

### Step 1 — Read the Approved Plan

Use the architecture plan from the `analyze-requirement` skill as the source of truth. Do not deviate from it without user confirmation.

### Step 2 — Look Up Providers and Resources

Use the Terraform MCP to:
- Fetch the latest provider versions
- Look up each resource type from the approved plan — fetch its exact required and optional arguments from the registry before writing that resource block
- Identify recommended modules
- Do not rely on training data for resource arguments — always verify against the registry

### Step 3 — Generate Project Structure

Use the project name from the approved plan as the folder name. The same project name must also be set as the `project_name` value in `terraform.tfvars`.

Create a dedicated project folder with the following files:

```
project-name/
├── providers.tf
├── versions.tf
├── variables.tf
├── main.tf
├── outputs.tf
├── terraform.tfvars
└── README.md
```

For larger deployments, split resources logically (e.g., `networking.tf`, `compute.tf`, `storage.tf`).

### Step 4 — Remote State

Always configure an S3 remote state backend using the bucket name and region gathered during `analyze-requirement`.

Terraform does not support variables in the backend block. Use literal values directly:

```hcl
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "project-name/terraform.tfstate"
    region = "us-east-1"
  }
}
```

Use the actual values from the approved plan. Do not use `var.*` references in the backend block.

### Step 5 — Variablize Everything

All configurable values must be defined as variables in `variables.tf`. No hardcoded values anywhere in the code.

Every variable must have:
- A `description`
- A `type`
- A `default` only where a safe, sensible default exists

Common variables to always include:
- `aws_region`
- `environment`
- `project_name`
- `state_bucket`

Generate a `terraform.tfvars` file in the project folder with the actual values collected during `analyze-requirement`. This file is the single source of truth for variable values and is used by the pipeline at runtime.

Example:
```hcl
aws_region   = "us-east-1"
environment  = "dev"
project_name = "vpc-ec2"
state_bucket = "my-terraform-state"
```

### Step 6 — Terraform Standards

- Pin provider versions
- Use `locals` for repeated values
- Tag all resources using `environment` and `project_name` variables
- Follow least-privilege IAM
- Document all variables and outputs

### Step 7 — Format Before Committing

All generated Terraform files must follow canonical Terraform formatting before being committed. Apply `terraform fmt` style rules to every `.tf` and `.tfvars` file:
- Consistent indentation (2 spaces)
- Aligned `=` signs within blocks
- No trailing whitespace
- One blank line between blocks

Do not commit files that would fail `terraform fmt -check`.

### Step 8 — Commit Files and Raise PR

Use the GitHub MCP to:
1. Create a feature branch named `terraform/{project_name}`
2. Commit all generated files into the project folder on that branch
3. Raise a pull request to main with a clear description of the infrastructure being created
