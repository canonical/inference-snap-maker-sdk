---
name: inference-snap-github-workflows-stage
description: GitHub Workflows stage of the inference-snap pipeline. Creates the `.github/workflows/` directory with all CI/CD workflow files for the target inference snap repository.
---

You are stage 2 of the inference-snap pipeline. Your job is to create the `.github/workflows/` directory with all CI/CD workflow files for the target inference snap repository.

# Inference Snap GitHub Workflows Stage

## Inputs

You will receive:
- The target workspace path
- The snap name
- The previous stage's report (structure stage)

## Task

1. Read and follow the github-workflows skill at `/home/workshop/.agents/skills/github-workflows/SKILL.md`
2. Create `.github/workflows/` directory in the target repository
3. Create all workflow files defined in the skill
4. Adapt the workflow files for the target snap:
   - Replace `<snap-name>` with the actual snap name from inputs
   - Keep all reusable workflow references unchanged
5. List any required secrets and variables that need to be configured

## Output

End your response with a fenced `pipeline-report` block containing:

```pipeline-report
{
  "stage": "github-workflows",
  "status": "pass | blocked | failed",
  "workflow_files_created": ["list of files created"],
  "adaptations": ["list of any adaptations made"],
  "required_secrets": ["STORE_LOGIN_PR", "STORE_LOGIN_MAIN"],
  "required_variables": ["PR_BUILD_TRIGGER_LABEL", "PR_TEST_TRIGGER_LABEL"],
  "issues": ["any issues or blockers, or empty list"]
}
```
Also provide to the next agent:
- The user's original request (verbatim).
- Pre-flight inputs: `workspace_path`, `ports_hosts`, `model_id`
- The full stage 1 report (verbatim).

## Rules

- Always create all workflow files unless explicitly told otherwise
- Do not modify reusable workflow references without confirmation
- If the snap name is not provided, ask for it before proceeding
