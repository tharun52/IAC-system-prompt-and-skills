---
name: ma-gitlab-fix-pipeline
description: "Fetch pipeline logs, diagnose the failure, fix the Terraform code that caused the error, and raise or update the MR. Never modify pipeline files — only fix terraform/ and env/ files."
---

## Pipeline stages (MR pipeline)

```
format       → terraform_fmt
security     → terraform_init       (artifacts: .terraform/, .terraform.lock.hcl)
validate     → terraform_validate   (needs: terraform_init)
             → checkov_scan         (needs: terraform_init, parallel with validate)
plan         → terraform_plan       (needs: terraform_init + terraform_validate + checkov_scan)
```

## Workflow

### Step 1 — Fetch the Logs

Use the GitLab MCP to fetch the latest failed pipeline for the branch:
1. Call `list_pipelines` twice in parallel:
   - With `ref: terraform/{project_name}` and `status: failed` — catches branch/push pipelines
   - With `source: merge_request_event` and `status: failed` — catches MR pipelines
   Pick the most recent pipeline ID across both results.
2. Call `list_pipeline_jobs` with that pipeline ID — identify the failed job name and ID
3. Call `get_pipeline_job_output` with the failed job ID — extract the exact error message

### Step 2 — Diagnose

Identify the root cause by failed job:

| Failed job | Likely cause in Terraform code |
|---|---|
| `terraform_fmt` | `.tf` file has wrong indentation, misaligned `=` signs, inline comments after values, or tabs |
| `terraform_init` | Wrong backend config in `backend.tf`, missing provider source in `versions.tf` |
| `terraform_validate` | Invalid resource argument, missing variable, wrong reference in a `.tf` file |
| `checkov_scan` | Security misconfiguration in a `.tf` file (soft-fail — should not block) |
| `terraform_plan` | Resource argument error or wrong value in a `.tf` or `.tfvars` file |

If the failure is an **IAM permission error** — escalate to the orchestrator, do not attempt a fix.

### Step 3 — Diagnose Which Files Need Fixing

From the error output, identify every file that needs a change. Do not read files first — `edit_files` fetches them internally.

### Step 4 — Fix the Terraform Code

Use `edit_files` to apply all fixes in a single commit. Pass only the exact search and replace strings — never the full file content. All files are committed atomically.

```json
{
  "project_id": "{gitlab_project_path}",
  "branch": "terraform/{project_name}",
  "commit_message": "fix: {root cause summary}",
  "files": [
    {
      "file_path": "terraform/networking/main.tf",
      "changes": [
        { "search": "<exact broken string>", "replace": "<fixed string>" }
      ]
    }
  ]
}
```

Fix only the Terraform code that caused the error. Do not change anything else.

Common fixes:

| Error | Terraform fix |
|---|---|
| `fmt` failure | Fix indentation, align `=` signs, remove inline comments after values in the flagged `.tf` file |
| Invalid resource argument | Use `get_provider_details` from Terraform MCP to get the correct argument, fix the resource block |
| Missing variable | Add to `variables.tf` and `terraform.tfvars` |
| Wrong variable value | Update `terraform/terraform.tfvars` and `env/{env}.tfvars` |
| Wrong backend config | Fix `backend.tf` — bucket, key, region, or role_arn |

### Step 5 — Raise or Update the MR

Check if an MR already exists for the branch:
- Call `list_merge_requests` with `source_branch: terraform/{project_name}` and `state: opened`
- **If an MR exists** — call `update_merge_request` to update its description with the fix details
- **If no MR exists** — call `create_merge_request` with `source_branch: terraform/{project_name}` and `target_branch: main`

MR description:
```
## Fix: {root cause summary}

**Error:** {exact error from logs}

**Root cause:** {what caused it}

**Change:** {what was fixed and in which Terraform file}
```

Return the MR link to the orchestrator.
