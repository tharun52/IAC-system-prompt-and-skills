pipeline-context

Provides context on how the generic Terraform pipeline works, how it is triggered, and what the agent should communicate to the user after Terraform code is committed.


# Skill: Pipeline Context

## Objective

There is a single generic GitHub Actions pipeline at `.github/workflows/terraform.yml` shared across all Terraform projects. This skill is not for generating a new pipeline — it is for understanding how it works and guiding the user on how to use it.

## Pipeline Structure

### Triggers

Manual only via `workflow_dispatch`. The pipeline never runs automatically on push or PR.

Two inputs:
- `project_path` — the folder name of the project to target (e.g., `vpc-ec2`)
- `action` — `apply` or `destroy`

### Jobs

**1. run** — single job, always runs fmt, init, validate, plan first, then:
- If `action=apply`: runs `terraform apply tfplan`
- If `action=destroy`: runs `terraform destroy -var-file=terraform.tfvars -auto-approve`
- Uploads plan artifact (retained 30 days)

### Variables

All Terraform variables are sourced from `terraform.tfvars` inside the project folder. No variables are passed via GitHub environment variables or secrets — only AWS credentials come from the `Aws_runner` GitHub Environment.

### AWS Credentials

Sourced from the `Aws_runner` GitHub Environment secrets:
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `AWS_SESSION_TOKEN`

### Other Settings
- Terraform plugins cached per project using `.terraform.lock.hcl` hash
- Concurrency protection prevents parallel runs on the same project

## What to Tell the User After Code is Committed

After the Terraform code and `terraform.tfvars` are committed and the PR is raised, inform the user:

1. **Review the PR** — no pipeline runs automatically. The PR is purely for code review.
2. **To deploy**, go to GitHub Actions → select the Terraform workflow → click "Run workflow" → enter the project folder name and select `apply`.
3. **To destroy**, same as above but select `destroy` as the action.
4. **AWS credentials** must be configured in the `Aws_runner` GitHub Environment if not already set up.
