---
name: inference-snap-build-and-test
description: Build, install, and runtime-verify an inference snap including engine selection and prompt/API checks.
trigger: Keywords like "build check", "pack snap", "install and test", "prompt test", "runtime verification"
scope: user
---

# Inference Snap Build And Test

## Purpose

Execute end-to-end verification that the snap builds, installs, starts, and serves prompts correctly.

## Host prerequisites (check before building)

`snapcraft pack --destructive-mode` builds directly on the host, so verify these
first and stop-and-report anything missing rather than silently degrading:

- **OS:** Ubuntu matching the snap `base` (e.g. `core24` ⇒ noble 24.04).
- **Tools:** `snapcraft`, `snapd` active, `make`, `cmake`, `build-essential`,
  `git`, plus smoke-test deps `curl`, `jq`, `ss` (iproute2). Install any missing.
- **Root:** `--destructive-mode` installs build-packages, so run it under `sudo`.
- **Disk:** compute the summed size of the model + all shard/runtime components
  and require that much free space **on the filesystem backing
  `/var/lib/snapd/snaps`** (where `snap install` copies artifacts), not just the
  build dir. A large model can exceed a small container root FS; if so, stop and
  report that the root FS must be resized (or relocate snapd storage only with
  explicit approval).
- **GPU:** detect `/dev/nvidia*` / `nvidia-smi` / `/dev/dri`. A GPU engine can
  only be smoke-tested if the matching hardware is present; otherwise it will
  report `compatible: false` and is skipped (not a failure).

## Model preparation (before packing)

The model artifacts are not committed. Before `snapcraft pack`, run the repo's
model preparation (typically `./download-models.sh`, which may wrap
`make download-models && make split-model`) so the component payloads exist on
disk. For sharded models, confirm every shard landed in its component directory
with the exact filename the `model.yaml` layout expects.

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
