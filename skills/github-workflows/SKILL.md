---
name: github-workflows
description: Create GitHub Actions workflow files for inference snap repositories based on canonical/gemma4-snap patterns.
trigger: Keywords like "add github workflows", "create CI/CD workflows", "set up github actions", "add workflow files"
scope: user
---

# GitHub Workflows for Inference Snaps

## Purpose

Create the `.github/workflows/` directory with proven CI/CD workflow files for inference snap repositories. These workflows handle building, testing, CLA checking, and engine validation.

## Workflow Files

### 1. build-and-test-pr.yaml

Triggered on PR labels for building and testing pull requests.

```yaml
name: Build+Test PR

on:
  pull_request:
    types: [ labeled ]

jobs:
  remove-build-label:
    permissions:
      contents: read
      pull-requests: write
    uses: canonical/inference-snaps-dev/.github/workflows/remove-label.yaml@v2
    with:
      label: ${{ vars.PR_BUILD_TRIGGER_LABEL }}

  remove-test-label:
    permissions:
      contents: read
      pull-requests: write
    uses: canonical/inference-snaps-dev/.github/workflows/remove-label.yaml@v2
    with:
      label: ${{ vars.PR_TEST_TRIGGER_LABEL }}

  build-pre-check:
    if: ${{ github.event.label.name == vars.PR_TEST_TRIGGER_LABEL }}
    runs-on: ubuntu-latest
    outputs:
      build-success: ${{ steps.build-pre-check.outputs.build-success }}
    steps:
      - id: build-pre-check
        name: Check for existing successful build
        uses: canonical/inference-snaps-testing/.github/actions/build-pre-check@v1
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          workflow-name: ${{ github.workflow }}

  build-and-publish:
    needs: [build-pre-check]
    if: always() && ((github.event.label.name == vars.PR_BUILD_TRIGGER_LABEL) || (github.event.label.name == vars.PR_TEST_TRIGGER_LABEL && needs.build-pre-check.outputs.build-success == 'false'))
    uses: canonical/inference-snaps-dev/.github/workflows/build-publish-snap.yaml@v2
    strategy:
      fail-fast: false
      matrix:
        runner:
          - [ self-hosted, amd64, noble, 2xlarge-extra ]
          - [ self-hosted, arm64, noble, large-extra ]
    with:
      runner: "${{ toJSON(matrix.runner) }}"
      init-build-script: 'download-models.sh'
      publish-channel: latest/edge/pr-${{ github.event.pull_request.number }}
    secrets:
      store-credentials: ${{ secrets.STORE_LOGIN_PR }}

  test:
    needs: [build-and-publish]
    if: ${{ always() && github.event.label.name == vars.PR_TEST_TRIGGER_LABEL && (needs.build-and-publish.result == 'success' || needs.build-and-publish.result == 'skipped') }}
    uses: ./.github/workflows/testflinger-tests.yaml
    with:
      snap-channel: latest/edge/pr-${{ github.event.pull_request.number }}
```

### 2. build-main.yaml

Triggered on pushes to the main branch.

```yaml
name: Build main

on:
  push:
    branches: [ main ]

jobs:
  build-and-publish:
    uses: canonical/inference-snaps-dev/.github/workflows/build-publish-snap.yaml@v2
    strategy:
      fail-fast: false
      matrix:
        runner:
          - [ self-hosted, amd64, noble, 2xlarge-extra ]
          - [ self-hosted, arm64, noble, large-extra ]
    with:
      runner: "${{ toJSON(matrix.runner) }}"
      init-build-script: 'download-models.sh'
      publish-channel: 'latest/edge' # Add track if needed, e.g., 24/latest/edge
    secrets:
      store-credentials: ${{ secrets.STORE_LOGIN_MAIN }}
```

### 3. cla-check.yaml

Checks if the CLA has been signed on pull requests.

```yaml
name: cla-check
on: [pull_request]

jobs:
  cla-check:
    runs-on: ubuntu-slim
    steps:
      - name: Check if CLA signed
        uses: canonical/has-signed-canonical-cla@v2
```

### 4. testflinger-tests.yaml

Reusable workflow for running Testflinger tests against the snap.

```yaml
name: Testflinger tests

on:
  workflow_call:
    inputs:
      snap-channel:
        description: Snap channel to test
        required: true
        type: string
  workflow_dispatch:
    inputs:
      snap-channel:
        description: Snap channel to test
        required: true
        type: string

jobs:
  test-snap:
    name: ${{ matrix.jobs.select-engine }} engine
    strategy:
      fail-fast: false
      matrix:
        jobs:
          - job-queue: maas-systemtests-amd64
            provision-data: "distro: noble"
            select-engine: cpu
            test-chat-tps: true
            test-image-prompt: true

          - job-queue: anbox-nvidia-amd64
            provision-data: "distro: noble"
            install-nvidia-driver-version: 595
            select-engine: nvidia-gpu
            test-chat-tps: true
            expected-tps: 9.4
            test-image-prompt: true

          # - job-queue: anbox-amd-amd64
          #   provision-data: "distro: noble"
          #   select-engine: amd-gpu
          #   test-chat-tps: true
          #   expected-tps: 10
          #   test-image-prompt: true

    uses: canonical/inference-snaps-testing/.github/workflows/test-snap.yaml@v1
    secrets: inherit
    with:
      job-queue: ${{ matrix.jobs.job-queue }}
      provision-data: ${{ matrix.jobs.provision-data }}

      snap-name: <snap-name>
      snap-channel: ${{ inputs.snap-channel }}

      install-nvidia-driver-version: ${{ matrix.jobs.install-nvidia-driver-version }}
      install-intel-npu-driver: ${{ matrix.jobs.install-intel-npu-driver == true }}
      snap-connections: ${{ matrix.jobs.snap-connections }}

      expected-engine: ${{ matrix.jobs.expected-engine }}
      select-engine: ${{ matrix.jobs.select-engine }}

      test-chat-tps: ${{ matrix.jobs.test-chat-tps == true }}
      expected-tps: ${{ matrix.jobs.expected-tps }}

      test-image-prompt: ${{ matrix.jobs.test-image-prompt == true }}
      devmode: ${{ matrix.jobs.devmode == true }}
```

### 5. validate-engines.yaml

Validates engine manifests on pull requests.

```yaml
name: Validate engine manifests
on: [pull_request]

jobs:
  validate:
    runs-on: [ ubuntu-latest ]
    steps:

      - name: Checkout code
        uses: actions/checkout@v6

      - name: Checkout inference-snap-cli
        uses: actions/checkout@v6
        with:
          repository: canonical/inference-snaps-cli
          path: inference-snaps-cli
          ref: v1.0.0

      - name: Set up Go
        uses: actions/setup-go@v6
        with:
          go-version: '1.26.x'

      - name: Run validation
        run: |
          cd inference-snaps-cli/cmd/cli
          go run . debug validate-engines ../../../engines/**/*.yaml
```

### 6. init-build.sh

Helper script used during builds to prune LFS objects.

```bash
#!/bin/bash -ex

# Remove LFS object copies to reduce disk usage
git lfs prune --force
```

## Workflow

1. Create `.github/workflows/` directory in the target repository.
2. Create all workflow files listed above, adapting:
   - `snap-name` in testflinger-tests.yaml to match the target snap name
   - `init-build-script` in build workflows if a different script is needed
   - `publish-channel` values if different channels are required
   - Matrix jobs in testflinger-tests.yaml based on target engines and test infrastructure
3. Create `init-build.sh` in the workflows directory and make it executable.
4. Verify that required secrets are configured in the repository:
   - `STORE_LOGIN_PR` — credentials for publishing PR builds
   - `STORE_LOGIN_MAIN` — credentials for publishing main branch builds
5. Verify that required variables are configured in the repository:
   - `PR_BUILD_TRIGGER_LABEL` — label that triggers PR builds
   - `PR_TEST_TRIGGER_LABEL` — label that triggers PR tests

## Output

- List of workflow files created.
- Any adaptations made from the reference workflows.
- Reminder to configure required secrets and variables.

## Rules

- Always create all workflow files unless explicitly told otherwise.
- Replace `<snap-name>` in testflinger-tests.yaml with the actual snap name.
- Do not modify reusable workflow references (canonical/inference-snaps-dev, canonical/inference-snaps-testing) without confirmation.
- Keep commented-out matrix entries (e.g., AMD GPU) for reference.
- Ensure init-build.sh has executable permissions.
