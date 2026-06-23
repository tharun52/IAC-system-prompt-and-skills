pipeline-monitor

Monitor a triggered GitHub Actions pipeline, report its status, diagnose failures, and return the diagnosis for fixing.

## Objective

Watch a GitHub Actions pipeline run, report progress, and if it fails, diagnose the root cause from the logs and return the findings. Do not fix anything — pass the diagnosis back to the IAC agent to handle.

## Workflow

### Step 1 — Get the Pipeline Run

Use the GitHub MCP to fetch the latest workflow run:
- Identify the correct workflow file based on what was triggered (`feature.yml`, `merge.yml`, or `manual.yml`)
- Report the current status to the user (queued, in progress, succeeded, failed)

### Step 2 — Poll Until Complete

Continue checking the run status until it reaches a terminal state. Report each job's status as it completes:

**`feature.yml`:** `format` → `validate` + `checkov` (parallel) → `plan`

**`merge.yml`:** single `deploy` job (init → plan → apply)

**`manual.yml`:** single `manual` job (fmt → init → validate → plan → apply or destroy)

### Step 3 — On Success

Report a success summary:
- Which jobs ran
- Any plan output worth highlighting (resources to add, change, destroy)
- Whether any fixes were applied during this monitoring session (so the agent knows whether to update the PR description)

If the run included an apply (`merge.yml` or `manual.yml` with `action: apply`), proceed to Step 3a.

#### Step 3a — Verify Deployment (apply only)

Use the AWS MCP to confirm the resources were actually created in AWS:
- Look up each key resource from the plan (e.g., VPC, EC2 instance, S3 bucket)
- Confirm they exist and are in the expected state (e.g., `available`, `running`)
- Report the actual resource IDs and states back

If a resource is missing or in an unexpected state despite the pipeline succeeding, report it clearly.

### Step 4 — On Failure

When a job fails:

1. Use the GitHub MCP to fetch the full logs for the failed job.
2. To read the Terraform files, use `get_file_contents` with the exact file path and the relevant branch ref (e.g., `terraform/{project_name}` or `main`). This returns the full file content.
3. Identify the root cause from the logs and file content (e.g., fmt check failed, invalid resource argument, missing variable, provider version mismatch, syntax error, IAM permission issue).
4. Determine which file needs to be fixed:
   - `fmt` failure → the specific `.tf` or `.tfvars` file with formatting issues (misaligned `=`, wrong indentation, missing blank lines)
   - Terraform code issue → relevant `.tf` file in `terraform/`
   - Wrong or missing variable value → `terraform/terraform.tfvars` and the corresponding `env/{env}.tfvars`
   - Pipeline issue → the relevant workflow file in `.github/workflows/`
   - Missing AWS permission → escalate to the user, do not attempt a fix
5. Return the following:
   - The exact error message from the logs
   - The identified root cause
   - Which file needs to be fixed and what needs to change

Do not attempt to fix anything. Pass the diagnosis to the IAC agent.

### Step 5 — Escalate if Needed

If the root cause cannot be determined from the logs or requires user input (e.g., missing AWS permissions), escalate with the exact error and what is needed from the user.
