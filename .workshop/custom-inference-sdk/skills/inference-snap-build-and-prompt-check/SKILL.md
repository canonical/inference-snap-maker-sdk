---
name: inference-snap-build-and-prompt-check
description: Build, install, and runtime-verify an inference snap including engine selection and prompt/API checks.
trigger: Keywords like "build check", "pack snap", "install and test", "prompt test", "runtime verification"
scope: user
---

# Inference Snap Build And Prompt Check

## Purpose

Execute end-to-end verification that the snap builds, installs, starts, and serves prompts correctly.

## Required Workflow

1. Build and pack:
   - Run: snapcraft pack -v
   - Confirm .snap and .comp artifacts are generated.
2. Install artifacts:
   - Run: sudo snap install *.snap *.comp --dangerous
3. Connect required interfaces:
   - hardware-observe
   - opengl
   - network-bind
   - process-control
   - (any additional interfaces declared in snapcraft.yaml)
4. Hardware and engine verification:
   - modelctl show-hardware (or snap command equivalent)
   - modelctl use-engine --auto --assume-yes --verbose
   - modelctl show-engine
   - If auto-selection fails, set intended engine explicitly and re-check.
5. Runtime service checks:
   - Verify daemon status and logs.
   - Confirm server binds to configured host/port.
6. Prompt checks:
   - GET /v1/models returns expected model alias.
   - POST /v1/chat/completions succeeds with a short prompt.
   - Capture response and latency/high-level runtime notes.
7. Failure handling:
   - If any step fails, identify root cause, apply fix, rebuild/reinstall/retest.

## Output

- Build result with artifact names.
- Installation and interface connection status.
- Selected engine details.
- Prompt/API test result (success/failure with response snippet).
- Remaining risks or follow-ups.

## Rules

- Do not declare success without a real prompt/API response.
- Include exact failing command/output summary when blocked.
- Re-run the failed verification step after each fix.
