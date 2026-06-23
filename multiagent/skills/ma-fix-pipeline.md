ma-fix-pipeline

Fetch pipeline logs, diagnose the failure, fix the identified file on the existing `terraform/{project_name}` branch, and raise a PR.

## Workflow

### Step 1 — Fetch the Logs

Use the GitHub MCP to fetch the latest failed workflow run for the repo:
- Identify the failed job and step
- Fetch the full logs for that job
- Extract the exact error message

### Step 2 — Diagnose

Identify the root cause from the logs:
- `fmt` failure → which file has formatting issues
- Invalid resource argument → which resource and argument
- Missing variable → which variable is missing and where
- Provider/version issue → what version conflict exists
- IAM permission error → escalate to the orchestrator, do not attempt a fix

### Step 3 — Read the Current File

Use `get_file_contents` with `ref: terraform/{project_name}` to read the file identified in Step 2.

### Step 4 — Apply the Fix

Fix only what the diagnosis identifies. Do not change unrelated code.

- **Single file fix** → use `create_or_update_file`
- **Multiple files** → use `push_files` to commit all changes in one commit

Common fix types:
- `fmt` failure → fix indentation, align `=` signs, fix blank lines in the affected file
- Invalid resource argument → correct using Terraform MCP (`get_provider_details`) to verify the right value
- Missing variable → add to `variables.tf` and `terraform.tfvars`
- Wrong variable value → update `terraform/terraform.tfvars` and `env/{env}.tfvars`

### Step 5 — Raise or Update the PR

Check if a PR already exists for the `terraform/{project_name}` branch using the GitHub MCP:
- **If a PR exists** — update its description with the latest fix details
- **If no PR exists** — raise a new PR to main

PR body:

```
## Fix: {root cause summary}

**Error:** {exact error from logs}

**Root cause:** {what caused it}

**Change:** {what was fixed and in which file}
```

Return the PR link to the orchestrator.
