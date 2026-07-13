ma-gitlab-validate-requirement

Check existing AWS and GitLab state, then produce an architecture plan from the requirements provided by the orchestrator.

## Workflow

### Step 1 — Check Existing GitLab State

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
- **Folder and file structure** — always use subfolders, never a flat list of `.tf` files. Group resources by logical layer:
  - `networking/` — VPC, subnets, route tables, security groups
  - `compute/` — EC2, launch templates, ASG
  - `storage/` — S3, EFS
  - `iam/` — roles, policies, instance profiles
  - `alb/` — load balancer, target groups, listeners

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
    ├── networking/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── compute/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

Adjust subfolders to match the actual resources in the plan. Never return a flat file list like `s3.tf`, `asg.tf` — always map them into the subfolder structure above.
