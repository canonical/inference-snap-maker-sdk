---
name: inference-snap-build-and-test
description: Build, install, and runtime-verify an inference snap including engine selection and prompt/API checks.
trigger: Keywords like "build check", "pack snap", "install and test", "prompt test", "runtime verification"
scope: user
---

# Inference Snap Build And Test

## Purpose

Execute end-to-end verification that the snap builds, installs, starts, and serves prompts correctly.

## Required Workflow

1. Build and pack:
   - Run: snapcraft pack --destructive-mode
   - Do not use any other command to build or pack the snap, as it may not produce the expected artifacts.
2. Install artifacts:
   - Run: sudo snap install *.snap *.comp --dangerous
3. Connect required interfaces:
   - Run: sudo snap connect <snap-name>:<interface>
   - Required interfaces include:
      - hardware-observe
      - opengl
      - network-bind
      - process-control
      - Any additional interfaces declared in snapcraft.yaml.
4. Smoke test the snap:
   - The snap should already be running after installation on the host machine. Check with: sudo <snap-name> status.
   - Clone the repository https://github.com/canonical/inference-snaps-dev and switch to branch v2
   - Use the `smoke-test.sh` script to verify the snap.
   - Confirm the script output indicates successful execution.
   - If the script fails, report the exact failing command and output, and apply a patch with the smallest viable fix
   - If the script passes, proceed to the next step.
   - Discover which are compatible by running `<snap-name> list-engines --format json`. Check the key compatible in the output for each engine.
   - Repeat the smoke test for each compatible engine. 

## Rules

- Do not declare success without a real smoke test pass.
- Include exact failing command/output summary when blocked.
- Re-run the failed verification step after each fix.
- Do not change the testing script `smoke-test.sh`.
