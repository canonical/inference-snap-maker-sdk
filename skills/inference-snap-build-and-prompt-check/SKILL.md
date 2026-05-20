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
   - Run: ./dev/build.sh
   - Confirm .snap and .comp artifacts are generated.
2. Install artifacts:
   - Run: ./dev/install.sh
   - Confirm command output indicates successful installation.
3. Verify snap status:
   - Run: <snap-name> status
   - Confirm that every service is active
   - Confirm that the listed endpoints match the expected port.
   - Confirm validity of the model name.
4. Verify logs:
   - Run: sudo snap logs <snap-name> -n=100
   - Confirm no critical errors and that the server started successfully with the expected model.
5. Prompt checks:
   - GET /v1/models returns expected model alias.
   - POST /v1/chat/completions succeeds with a short prompt.
   - If the model has vision support, make a request with an image input and confirm expected response.
   - Capture response and latency/high-level runtime notes.
6. Failure handling:
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
