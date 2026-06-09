---
name: inference-snap-pipeline
description: Run the full inference-snap workflow as a sequential chain of subagents — structure → github workflows → static checks → build & prompt check → create PR — passing each stage's report as input to the next.
trigger: Keywords like "run the full inference snap pipeline", "chain inference snap skills", "run all inference-snap skills", "run each inference-snap skill in a subagent", "end-to-end inference snap"
scope: user
---

# Inference Snap Pipeline

## Purpose

Orchestrate the specialized inference-snap skills as a sequential chain of subagents. Each stage produces a structured report that becomes the input to the next stage.

`inference-snap-from-example` is a dispatcher and is NOT a pipeline stage.

## Stages (run sequentially — never in parallel)

1. **Structure** — follow `/home/workshop/.agents/skills/inference-snap-structure/SKILL.md`
2. **GitHub workflows** — follow `/home/workshop/.agents/skills/github-workflows/SKILL.md`
3. **Static checks** — follow `/home/workshop/.agents/skills/inference-snap-static-checks/SKILL.md`
4. **Build & prompt check** — follow `/home/workshop/.agents/skills/inference-snap-build-and-prompt-check/SKILL.md`
5. **Create PR** — `subagent_type: inference-snap-create-pr-stage`

## Pre-flight (before launching stage 1)

Collect from the user, then reuse across all stages:

- Target workspace path.
- Whether the target repo exists or must be created.
- Snap name.
- Ports/hosts for hooks.
- Model identity + whether it is > 5 GB (drives sharding decision).
- Model download URL.
- Remote repo URL (GitHub repository URL for the final PR, e.g. `https://github.com/org/repo`) explicitly confirmed by the user in the current run. Do not infer from existing git remotes.

If any of these are missing, ask before starting the chain.

## Orchestration rules

- Use the `Agent` tool with dedicated `subagent_type` values for each stage. Do NOT use `general-purpose`:
  - Stage 1: `inference-snap-structure-stage` (agent file: `/home/workshop/.agents/agents/inference-snap-structure-stage.md`)
  - Stage 2: `inference-snap-github-workflows-stage` (agent file: `/home/workshop/.agents/agents/inference-snap-github-workflows-stage.md`)
  - Stage 3: `inference-snap-static-checks-stage` (agent file: `/home/workshop/.agents/agents/inference-snap-static-checks-stage.md`)
  - Stage 4: `inference-snap-build-prompt-check-stage` (agent file: `/home/workshop/.agents/agents/inference-snap-build-prompt-check-stage.md`)
  - Stage 5: `inference-snap-create-pr-stage` (agent file: `/home/workshop/.agents/agents/inference-snap-create-pr-stage.md`)
- Launch stages one at a time. Wait for stage N to return before launching stage N+1.
- Each subagent prompt MUST be self-contained — the subagent has no view of this conversation. Always include:
  1. The user's original request (verbatim).
  2. The pre-flight inputs above.
  3. The previous stage's full report (verbatim), or `N/A (first stage)`.
- The workflow, rules, and output contract are baked into the agent's system prompt; do not restate them in the user prompt and do not tell the agent to read a SKILL.md.
- After each stage returns, surface a one-line status to the user and check abort conditions before continuing.

## Per-stage prompt template

```
User's original request:
  {verbatim}

Pre-flight inputs:
  workspace_path: {...}
  repo_exists: {...}
  snap_name: {...}
  ports_hosts: {...}
  model_id: {...}
  model_download_url: {...}
  model_over_5gb: {...}
  remote_repo_url: {...}

Previous stage report:
  {verbatim previous report, or "N/A (first stage)"}
```

## Abort conditions

- Stage 3 reports any blocking issue → STOP. Surface the fix list and ask the user whether to fix and rerun stage 3, or abort.
- Stage 4 reports a failed prompt/API check or build failure → STOP. Surface the failing step + command output verbatim.
- Stage 5 reports a push failure, PR creation failure, or missing `trigger-tests` label → STOP. Surface the exact error and PR URL if partially created.
- Subagent returns without a `pipeline-report` block → STOP and ask the user how to proceed; do not fabricate the missing report.

## Final output to user

- One-line status per stage (pass / blocked / failed).
- The verbatim stage-5 report (includes PR URL and label confirmation).
- Remaining risks and follow-ups aggregated across stages.

## Rules

- Do NOT skip stages, even if the user seems to imply only the last is needed — earlier stages produce the inputs the later ones rely on.
- Do NOT run stages in parallel.
- Do NOT invoke `inference-snap-from-example` as part of the chain.
- Do NOT declare the pipeline successful unless stage 4 reports a real prompt/API response AND stage 5 confirms the PR with the `trigger-tests` label.
