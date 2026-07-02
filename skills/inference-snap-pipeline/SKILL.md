---
name: inference-snap-pipeline
description: Run the full inference-snap workflow as a sequential chain of subagents — structure → github workflows → static checks → build & test — passing each stage's report as input to the next.
trigger: Keywords like "start packing pipeline", "run inference snap pipeline", "create inference snap", "build inference snap", "inference snap workflow"
scope: user
---

# Inference Snap Pipeline

## Purpose

Orchestrate the specialized inference-snap skills as a sequential chain of subagents. Each stage produces a structured report that becomes the input to the next stage.


## Stages (run sequentially — never in parallel)

1. **Structure** — follow `skills/inference-snap-structure/SKILL.md`
2. **GitHub workflows** — follow `skills/github-workflows/SKILL.md`
3. **Static checks** — follow `skills/inference-snap-static-checks/SKILL.md`
4. **Build & Test** — follow `skills/inference-snap-build-and-test/SKILL.md`

## Pre-flight (before launching stage 1)

Parse from the `README.md` of the template repository directory where the `workshop.yaml` resides, then reuse across all stages:
- Snap name.
- Ports/hosts for hooks.

Instead assume the following:
- Target workspace path is where the `workshop.yaml` resides, it is the root directory of the inference snap repository
- The target repository is the `origin remote` of the git repository in the target workspace path. You can verify this with `git remote -v` and parse the `origin` URL.

Before starting the chain, make sure that all the previous inputs are available and valid, moreover make sure that there is a valid Makefile in the target workspace path.
It is there to download (and, for large models, split) the models. Subagents will need to use it.

The single model-preparation entrypoint used by both local builds and CI is a
repo-root **`download-models.sh`** (the CI reusable workflow invokes
`./download-models.sh`, not `make`). It SHOULD wrap the Makefile
(`make download-models && make split-model`). If the repo has a Makefile but no
`download-models.sh`, note that stage 2 must create the wrapper.

Also derive the model artifact URL (the HuggingFace `resolve` URL) from the
Makefile/README so stage 1 can measure its size and pick the sharding variant.

If any of these are missing, ask before starting the chain. Also prepare a recap and ask for confirmation before starting the chain.

## Orchestration rules

- Prefer the `Agent`/`Task` tool with dedicated `subagent_type` values for each stage. Do NOT use `general-purpose`:
  - Stage 1: `inference-snap-structure-stage` (agent file: `agents/inference-snap-structure-stage.md`)
  - Stage 2: `inference-snap-github-workflows-stage` (agent file: `agents/inference-snap-github-workflows-stage.md`)
  - Stage 3: `inference-snap-static-checks-stage` (agent file: `agents/inference-snap-static-checks-stage.md`)
  - Stage 4: `inference-snap-build-and-test-stage` (agent file: `agents/inference-snap-build-and-test-stage.md`)
- These `subagent_type`s must be registered in the opencode config (e.g. `opencode.jsonc`) for the `Task` tool to accept them. If the runtime rejects them as unknown agent types, do NOT fall back to `general-purpose`: instead execute each stage **inline yourself**, following the corresponding agent `.md` as your system prompt and its skill/RULESET as the reference, still producing the stage's `pipeline-report` block before moving on. Surface that you are running inline so the behavior is transparent.
- Launch stages one at a time. Wait for stage N to return (or complete inline) before launching stage N+1.
- Each subagent prompt MUST be self-contained — the subagent has no view of this conversation. Always include:
  1. The user's original request (verbatim).
  2. The pre-flight inputs above.
  3. The previous stage's full report (verbatim), or `N/A (first stage)`.
- The workflow, rules, and output contract are baked into the agent's system prompt; do not restate them in the user prompt.
- After each stage returns, surface a one-line status to the user and check abort conditions before continuing.

## Per-stage prompt template

```
User's original request:
  {verbatim}

Pre-flight inputs:
  workspace_path: {...}
  repo_origin: {...}
  snap_name: {...}
  ports_hosts: {...}
  model_id: {...}
  model_url: {HuggingFace resolve URL of the model artifact}

Previous stage report:
  {verbatim previous report, or "N/A (first stage)"}
```

## Abort conditions

- Stage 3 reports any blocking issue → STOP. Surface the fix list and ask the user whether to fix and rerun stage 3, or abort.
- Stage 4 reports a failed prompt/API check or build failure → STOP. Surface the failing step + command output verbatim.
- Subagent returns without a `pipeline-report` block → STOP and ask the user how to proceed; do not fabricate the missing report.

## Final output to user

- One-line status per stage (pass / blocked / failed).
- Remaining risks and follow-ups aggregated across stages.
- Link to open the PR in the target repository with a description to be used for the PR body.

## Rules

- Do NOT skip stages, even if the user seems to imply only the last is needed — earlier stages produce the inputs the later ones rely on.
- Do NOT run stages in parallel.
- Do NOT declare the pipeline successful unless stage 4 reports a successful smoke test run.
