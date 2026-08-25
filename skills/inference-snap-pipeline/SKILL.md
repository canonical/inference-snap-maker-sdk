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

Parse from the `README.md` of the template repository directory where the `workshop.yaml` resides, the inputs passed in the YAML frontmatter
block at the top of the file (between the opening and closing `---` markers) and verify that they are all present and valid. The frontmatter exposes `snap-name`, `snap-title`, `model-card`, `http-port`, `webui-http-port`, and `engines`. If any are missing or invalid, ask the user to provide them before starting the chain.

Assume the following:
- Target workspace path is where the `workshop.yaml` resides, it is the root directory of the inference snap repository
- The target repository is the `origin remote` of the git repository in the target workspace path. You can verify this with `git remote -v` and parse the `origin` URL.

Before starting the chain, make sure that all the previous inputs are available and valid, moreover make sure that there is a valid Makefile in the target workspace path.
It is there to download the models. Subagents will need to use it.

**Validate `snap-name` early.** The snap name MUST match
`^[a-z0-9]+(-[a-z0-9]+)*$` (snapd rule). If the input contains a dot, underscore, or uppercase (e.g. `qwen3.5`), it is invalid and `snapcraft pack` will fail late. Propose the hyphenated form (`qwen3.5` → `qwen3-5`), confirm with the user, and use it as the store name + CLI command; keep the original as the `snap-title` display name.

If any of these are missing, ask before starting the chain. Also prepare a recap and ask for confirmation before starting the chain.

After confirmation modify the README by replacing the `{...}` placeholders with the actual values. Do not modify any other part of the README.
Make sure that the engines list in the README is updated with the engines required by the user that set them in the frontmatter at the top of the README.

## Orchestration rules

- Prefer the `Agent`/`Task` tool with dedicated `subagent_type` values for each stage. Do NOT use `general-purpose`:
  - Stage 1: `inference-snap-structure-stage` (agent file: `agents/inference-snap-structure-stage.md`)
  - Stage 2: `inference-snap-github-workflows-stage` (agent file: `agents/inference-snap-github-workflows-stage.md`)
  - Stage 3: `inference-snap-static-checks-stage` (agent file: `agents/inference-snap-static-checks-stage.md`)
  - Stage 4: `inference-snap-build-and-test-stage` (agent file: `agents/inference-snap-build-and-test-stage.md`)
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

- Subagent returns without a `pipeline-report` block → STOP and ask the user how to proceed; do not fabricate the missing report.

## Final output to user

- One-line status per stage (pass / blocked / failed).
- Remaining risks and follow-ups aggregated across stages.
- Link to open the PR in the target repository with a description to be used for the PR body.

## Rules

- Do NOT run stages in parallel.
- Do NOT declare the pipeline successful unless stage 4 reports a successful smoke test run.
