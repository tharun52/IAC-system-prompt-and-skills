analyze-requirement

Gather requirements, clarify missing information, propose an architecture, and wait for user approval before implementation.

## Objective

Understand what the user wants to build, fill any gaps through clarifying questions, and produce an approved architecture plan before any code is written.

## Workflow

### Step 1 — Understand the Request

Read the user's request and identify:
- What AWS resources are needed
- The target environment (dev, staging, prod)
- Any stated preferences (region, naming, existing infrastructure)

### Step 2 — Ask Clarifying Questions

The following must always be asked — no exceptions:
- Project name (lowercase kebab-case, e.g. `vpc-ec2`) — used as the branch name and remote state key
- GitHub repository name (never create a new repo — always ask which existing repo to use)
- S3 bucket name for Terraform remote state
- Environment (dev / staging / prod) — determines which tfvars file is generated

Also ask if unclear:
- AWS region
- Resource sizing (instance type, storage, etc.)
- Networking (VPC CIDR, subnets, public/private)
- Security (IAM roles, security groups)
- Existing infrastructure to integrate with

Ask all questions in one message. Do not ask one at a time.

### Step 3 — Propose Architecture

Once you have enough information, present a clear architecture plan that includes:
- List of AWS resources to be created
- Key configuration decisions
- Any assumptions made

### Step 4 — Wait for Approval

Explicitly ask the user to approve the plan before proceeding.

Do not move to writing code until the user confirms.
