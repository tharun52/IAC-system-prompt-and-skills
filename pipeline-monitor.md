pipeline-monitor

Monitor a triggered GitHub Actions pipeline, report its status, and automatically diagnose and fix failures by updating the Terraform or workflow code and pushing a fix.


# Skill: Pipeline Monitor

## Objective

Watch a GitHub Actions pipeline run, report progress, and if it fails, diagnose the root cause, apply a fix, and re-trigger validation.

## Workflow

### Step 1 — Get the Pipeline Run

Use the GitHub MCP to fetch the latest workflow run for the project:
- Identify the workflow file (`terraform.yml`)
- Filter runs by the `project_path` input to get the run specific to the target project
- Report the current status to the user (queued, in progress, succeeded, failed)

### Step 2 — Poll Until Complete

Continue checking the run status until it reaches a terminal state (success or failure). There is a single job called `run` — report its status and each step as it completes:
- Terraform Format
- Terraform Init
- Terraform Validate
- Terraform Plan
- Terraform Apply or Terraform Destroy (depending on the action)

### Step 3 — On Success

Report a success summary to the user:
- Which jobs ran
- Any plan output worth highlighting (resources to add, change, destroy)

If the run was a `workflow_dispatch` apply, proceed to Step 3a to verify the deployment.

#### Step 3a — Verify Deployment (apply only)

Use the AWS MCP to confirm the resources were actually created in AWS:
- Look up each key resource from the plan (e.g., VPC, EC2 instance, S3 bucket)
- Confirm they exist and are in the expected state (e.g., `available`, `running`)
- Report the actual resource IDs and states back to the user

If a resource is missing or in an unexpected state despite the pipeline succeeding, report it clearly so the user can investigate.

### Step 4 — On Failure

When a job fails:

1. Use the GitHub MCP to fetch the full logs for the failed job.
2. Identify the root cause from the logs (e.g., invalid resource argument, missing variable, provider version mismatch, syntax error, IAM permission issue).
3. Report the error and root cause to the user clearly.
4. Determine where the fix needs to be applied:
   - Terraform code issue → fix the relevant `.tf` file
   - Wrong or missing variable value → fix `terraform.tfvars` in the project folder
   - Pipeline/workflow issue → fix `.github/workflows/terraform.yml`
   - Missing AWS permission → inform the user what IAM permission needs to be added
5. Apply the fix using the GitHub MCP to commit the updated file to the same branch.
6. Report to the user what was changed and why.

### Step 5 — Re-trigger

After pushing a fix, inform the user to re-trigger the pipeline manually via workflow_dispatch with the same `project_path` and `action` inputs.

### Step 6 — Escalate if Needed

If the root cause cannot be determined from the logs, or the fix requires user input (e.g., missing AWS permissions, unknown variable value), escalate clearly:
- Show the exact error
- Explain what information or action is needed from the user
- Do not guess or apply a blind fix

## Rules

- Never apply a fix without explaining what changed and why.
- If the fix changes the architecture (not just a syntax or config error), go back to the user for approval before applying.
