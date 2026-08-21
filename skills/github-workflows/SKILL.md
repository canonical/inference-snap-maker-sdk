---
name: github-workflows
description: Create GitHub Actions workflow files for inference snap repositories.
trigger: Keywords like "add github workflows", "create CI/CD workflows", "set up github actions", "add workflow files"
scope: user
---

# GitHub Workflows for Inference Snaps

## Purpose

Create the `.github/workflows/` directory with proven CI/CD workflow files for inference snap repositories. These workflows handle building, testing, CLA checking, and engine validation.

## Workflow Files

### 1. pr-label.yaml

Triggered on PR labels for building and testing pull requests.

```yaml
# This workflow is used to build and test pull requests when specific labels are applied.
# Supported labels: trigger-build, trigger-tests
name: PR Label

on:
  pull_request:
    types: [ labeled ]

concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number }}
  cancel-in-progress: true

permissions:
  contents: read
  pull-requests: write
  actions: read

jobs:
  build-test:
    name: ${{ github.event.label.name }}
    uses: canonical/inference-snaps-dev/.github/workflows/reuse-pr-build-test.yaml@v2
    with:
      trigger-label: ${{ github.event.label.name }}
      pr-number: "${{ github.event.pull_request.number }}"
      build-runner: |
        [
          ["amd64", "large", "noble", "self-hosted"],
          ["arm64", "noble", "self-hosted"]
        ]
      test-jobs-matrix: |
        [
          {
                    "job-queue": "maas-systemtests-amd64",
                    "provision-data": "distro: noble",
                    "select-engine": "cpu",
                    "test-chat-tps": true,
                    "test-image-prompt": true
          },
          {
                    "job-queue": "anbox-nvidia-amd64",
                    "provision-data": "distro: noble",
                    "install-nvidia-driver-version": 595,
                    "select-engine": "nvidia-gpu",
                    "test-chat-tps": true,
                    "expected-tps": 9.4,
                    "test-image-prompt": true
          }
        ]
      snap-name: {{SNAP_NAME}}
      testflinger-client-id: ${{ vars.TESTFLINGER_CLIENT_ID }}
    secrets:
      store-credentials: ${{ secrets.STORE_LOGIN_PR }}
      github-token: ${{ secrets.GITHUB_TOKEN }}
      testflinger-secret-key: ${{ secrets.TESTFLINGER_SECRET_KEY }}

```

### 2. validate-engines.yaml


```yaml
# This workflow is used to validate pull requests.
# It includes the Canonical CLA check and engine validation.
name: PR Checks
on: [pull_request]

permissions:
  contents: read
  pull-requests: read

jobs:
  cla:
    runs-on: ubuntu-slim
    steps:
      - name: Check if CLA signed
        uses: canonical/has-signed-canonical-cla@v2
        
  validate:
    runs-on: [ ubuntu-latest ]
    steps:

      - name: Checkout code
        uses: actions/checkout@v7
        with:
          # Prevent usage of simulated merge commit of PR
          ref: ${{ github.event.pull_request.head.sha || github.sha }}

      - name: Checkout inference-snap-cli 
        uses: actions/checkout@v7
        with:
          repository: canonical/inference-snaps-cli
          path: inference-snaps-cli
          ref: {{ CLI_TAG }}

      - name: Set up Go
        uses: actions/setup-go@v6
        with:
          go-version: '1.26.x'

      - name: Validate engines
        run: |
          cd inference-snaps-cli/cmd/modelctl
          go run . debug validate-engines ../../../engines/**/*.yaml

```

### 4. cicd.yaml

```yaml
# This workflow is used to build and publish the snap to the store on every push to main.
name: CICD

on:
  push:
    branches:
      - main
  
  workflow_dispatch: # manual trigger 


# Keep one CICD run active at a time so publish/test/promote happen in order
concurrency:
  group: ${{ github.workflow }}
  queue: max

jobs:
  cicd:
    if: github.repository == 'canonical/{{SNAP_REPOSITORY}}' # do not run on forks
    uses: canonical/inference-snaps-dev/.github/workflows/reuse-cicd.yaml@v2
    with:
      build-runner: |
        [
          ["amd64", "large", "noble", "self-hosted"],
          ["self-hosted", "arm64", "noble"]
        ]
      store-track: latest
      snap-name: {{SNAP_NAME}}
      smoke-test-engine: cpu
```

## Workflow

1. Create `.github/workflows/` directory in the target repository.
2. Create all workflow files listed above, adapting:
   - `snap-name` in testflinger-tests.yaml to match the target snap name
   - In `validate-engines.yaml`, replace `ref: {{ CLI_TAG }}` with the SAME
     `inference-snaps-cli` tag used by the `cli` part in `snap/snapcraft.yaml`
   - Set testflinger capability flags from the model's ACTUAL capabilities:
     `test-image-prompt` only for vision models; drop it (or set false) for
     text-only models. Only set `expected-tps` when you have a real measured
     baseline for that model+engine — do not copy another model's number.
   - Include one testflinger matrix job per engine the snap actually ships
     (e.g. `cpu`, `nvidia-gpu`); keep unrelated example jobs commented out.

## Output

- List of workflow files created.
- Any adaptations made from the reference workflows.
- Reminder to configure required secrets and variables.

## Rules

- Always create all workflow files unless explicitly told otherwise.
- Replace `<snap-name>` with the actual snap name.
- Pin `validate-engines.yaml`'s `inference-snaps-cli` ref to the same version as
  the `cli` part in `snap/snapcraft.yaml`; never leave the placeholder.
- Set `test-image-prompt`/`expected-tps` from the model's real capabilities and
  measured baselines — do not carry over another model's values.
- Do not modify reusable workflow references (canonical/inference-snaps-dev, canonical/inference-snaps-testing) without confirmation.
