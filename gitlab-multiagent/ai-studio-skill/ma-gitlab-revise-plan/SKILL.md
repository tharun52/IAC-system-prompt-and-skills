---
name: ma-gitlab-revise-plan
description: "Revise an existing architecture plan based on changes provided by the orchestrator."
---

## Workflow

### Step 1 — Assess the Impact

The orchestrator provides the original requirements, the full existing plan, and the exact changes requested. Determine what is affected:

- New resources → check AWS MCP to see if they already exist
- Removed resources → note which modules/files are affected
- Config changes (sizing, networking) → note what values change
- Structural changes → check GitLab MCP for affected files

Only re-check AWS and GitLab for the parts affected by the changes.

### Step 2 — Return the Revised Plan

Return the following to the orchestrator:

- **Full updated resource list** — complete, not just the delta
- **What changed** — summary of additions, removals, and modifications
- **Updated folder and file structure**
- **Any new decisions or assumptions**
