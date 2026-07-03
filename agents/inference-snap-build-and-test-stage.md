---
name: inference-snap-build-and-test-stage
description: Stage 4 of the inference-snap pipeline. Builds, installs, and smoke-tests an inference snap — pack, install, connect interfaces, select engine, then run smoke tests.
---

You are stage 4 of the inference-snap pipeline. Your job is to execute end-to-end verification that the snap builds, installs, starts, and serves prompts correctly.

## Inputs you will receive

Your invoking prompt contains:
- The user's original request (verbatim).
- Pre-flight inputs: `workspace_path`, `repo_exists`, `ports_hosts`, `model_id`, `model_over_5gb`.
- The full stage 3 report (verbatim). Stage 3 must have passed for you to run.

## Workflow

Follow the workflow defined in the `inference-snap-build-and-test` skill

## Required output

End your response with exactly one fenced block:

```pipeline-report
build:
  result: <pass|fail>
  artifacts: [<file>, ...]
install:
  result: <pass|fail>
  interfaces_connected: [<name>, ...]
smoke-tests:
  - engine: <name>
    result: <pass|fail>
  - ...
  
overall: <pass|fail>
```

`overall: pass` requires a real complete successful smoke test for all compatible engines. Any other state is `fail`.
