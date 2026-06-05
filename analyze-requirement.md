analyze-requirement

Gather requirements, clarify missing information, propose an architecture, and wait for user approval before implementation.


# Skill: Analyze Requirement

## Objective

Understand what the user wants to build, fill any gaps through clarifying questions, and produce an approved architecture plan before any code is written.

## Workflow

### Step 1 — Understand the Request

Read the user's request and identify:
- What AWS resources are needed
- The environment (dev, staging, prod)
- Any stated preferences (region, naming, existing infrastructure)

### Step 2 — Ask Clarifying Questions

If any of the following are unclear, ask the user before proceeding:
- Project name (lowercase kebab-case, e.g. `vpc-ec2`) — always required
- AWS region
- Environment (dev/staging/prod)
- Resource sizing (instance type, storage, etc.)
- Networking (VPC CIDR, subnets, public/private)
- Security (IAM roles, security groups)
- S3 state bucket name — always required
- Existing infrastructure to integrate with

Ask all questions in one message. Do not ask one at a time.

### Step 3 — Propose Architecture

Once you have enough information, present a clear architecture plan that includes:
- List of resources to be created
- Project folder name and file structure
- Key configuration decisions
- Any assumptions made

### Step 4 — Wait for Approval

Explicitly ask the user to approve the plan before proceeding.

Do not move to writing code until the user confirms.
