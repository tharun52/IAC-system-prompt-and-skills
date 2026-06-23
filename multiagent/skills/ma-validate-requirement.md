ma-validate-requirement

Check existing AWS and GitHub state, then produce an architecture plan from the requirements provided by the orchestrator.

## Workflow

### Step 1 — Check Existing GitHub State

Use `get_file_contents` (no `ref` — defaults to the repo's default branch) to check:
- `terraform/` folder — what `.tf` files and subfolders exist
- `env/` folder — what tfvars files exist

If the path does not exist, note that no existing Terraform code was found and continue.

Note what already exists so the plan does not duplicate it.

### Step 2 — Check Existing AWS State

Use the AWS MCP to check for resources relevant to the request in the target region. Note what already exists so the plan can reference it rather than recreate it.

### Step 3 — Return the Plan

Return the following to the orchestrator:

- **Resources to create**
- **Existing resources to integrate with** (if any found)
- **Existing repo state** (if any Terraform code found)
- **Key decisions and assumptions**
- **Folder and file structure**, for example:

```
repo/
├── env/
│   └── dev.tfvars
└── terraform/
    ├── versions.tf
    ├── providers.tf
    ├── variables.tf
    ├── outputs.tf
    ├── main.tf
    ├── terraform.tfvars
    ├── vpc/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── compute/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

Adjust subfolders to match the actual resources in the plan.
