pipeline-context

Context on how the Terraform pipelines work and what to tell the user after code is committed.

## Objective

There are three separate GitHub Actions pipelines for all Terraform work in the repo. This skill is not for generating pipelines — it is for understanding how they work so the Terraform code is written to be compatible with them and the user is guided correctly after code is committed.

## Pipelines

There are three separate workflow files in `.github/workflows/`:

**`feature.yml`** — triggers on push to any non-main branch (`terraform/**`):
`format` → `validate` + `checkov` (parallel) → `plan` (runs only after both pass)

**`merge.yml`** — triggers on PR merged to main (`terraform/**`):
Single `deploy` job: init → plan → apply

**`manual.yml`** — triggers on `workflow_dispatch`:
Single `manual` job: fmt + init + validate + plan → apply or destroy
Input: `action` — `plan` (default), `apply`, or `destroy`

## Key Facts

- All jobs run with `working-directory: terraform`
- Variables sourced from `terraform/terraform.tfvars`
- AWS credentials from the `Aws_runner` GitHub Environment

## What to Tell the User After the PR is Raised

1. Pushing to the branch runs fmt, validate, checkov, and plan automatically — fix any failures before raising a PR
2. Merging the PR runs plan and apply automatically
3. For a manual plan, apply, or destroy, go to GitHub Actions → Manual workflow → Run workflow → select the action (`plan` is the default)
4. Ensure AWS credentials are set in the `Aws_runner` GitHub Environment
