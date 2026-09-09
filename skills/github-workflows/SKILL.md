---
name: github-workflows
description: Create GitHub Actions workflow files for inference snap repositories based on canonical/gemma4-snap patterns.
trigger: Keywords like "add github workflows", "create CI/CD workflows", "set up github actions", "add workflow files"
scope: user
---

# GitHub Workflows for Inference Snaps

## Purpose

Modify the workflows contained inside the `.github/workflows/` directory. These workflows handle building, testing, CLA checking, and engine validation.

## Workflow

Modify each workflow file in the `.github/workflows/` directory as follows:
  - remove the #TODO comments after you address them
  - substitute `<snap-name>` with the actual snap name
  - substitute `<inference-snap-repository>` with the actual repository name
  - In `pr-checks.yaml`, make sure that the `lint` job is using the same
    `inference-snaps-cli` tag used by the `cli` part in `snap/snapcraft.yaml`
  - Set testflinger capability flags from the model's ACTUAL capabilities:
    `test-image-prompt` only for vision models; drop it (or set false) for
    text-only models. Only set `expected-tps` when you have a real measured
    baseline for that model+engine — do not copy another model's number.
  - Include one testflinger matrix job per engine the snap actually ships
    (e.g. `cpu`, `nvidia-gpu`); keep unrelated example jobs commented out.

## Output

- Any adaptations made from the reference workflows.

## Rules

- Replace `<snap-name>` with the actual snap name.
- Replace `<inference-snap-repository>` with the actual repository name.
- Pin `pr-checks.yaml`'s `inference-snaps-cli` ref to the same version as
  the `cli` part in `snap/snapcraft.yaml`; never leave the placeholder.
- Set `test-image-prompt`/`expected-tps` from the model's real capabilities and
  measured baselines — do not carry over another model's values.
- Do not modify reusable workflow references (canonical/inference-snaps-dev, canonical/inference-snaps-testing) without confirmation.
- Keep commented-out matrix entries (e.g., AMD GPU) for reference.
- Ensure init-build.sh has executable permissions.
