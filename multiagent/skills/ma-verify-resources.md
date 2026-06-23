ma-verify-resources

Verify that AWS resources from an approved plan were actually created after a successful deployment.

## Workflow

### Step 1 — Read the Context

The orchestrator provides:
- AWS region
- List of resources that were supposed to be created (from the approved plan)

### Step 2 — Check Each Resource

Use the AWS MCP to look up each resource by type. For each one check:
- Does it exist?
- Is it in the expected state (e.g. `available`, `running`, `active`)?
- Does its configuration match what was planned (e.g. instance type, subnet, tags)?

Common checks by resource type:
- **VPC / Subnets** → exists, state `available`, correct CIDR
- **EC2 / ASG** → instance exists, state `running`, correct type
- **RDS** → instance exists, state `available`, correct engine/size
- **S3** → bucket exists, correct region, versioning/encryption as configured
- **IAM roles** → role exists, correct policies attached
- **ALB / Target groups** → exists, state `active`, correct listeners

### Step 3 — Return the Report

Return the following to the orchestrator:

- **Verified resources** — resource ID, type, and current state for each resource that exists and is healthy
- **Missing resources** — any resource from the plan that could not be found in AWS
- **Unexpected state** — any resource that exists but is not in the expected state, with the actual state

If all resources are verified and healthy, state that clearly.
