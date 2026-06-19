# Inference Snaps SDK for Workshop

This SDK provides inference snap authoring and validation skills for Workshop environments, together with the OpenCode CLI. It includes specialized skills for snap structure scaffolding, static validation, build and prompt verification, GitHub workflow generation, and pull request creation. It also includes a pipeline skill that orchestrates these stages in sequence.

## Using the SDK

### 1. Reference workshop

```yaml
# workshop.yaml
name: inference-snap-dev
base: ubuntu@24.04
sdks:
  # Useful for testing the SDK in isolation, but not required to run the skills
  - name: vscode-remote
  - name: opencode
    channel: latest/stable
    # The SDK should be cloned inside the workshop directory and built with `sdkcraft try` before launching the workshop
  - name: try-inference-snaps-sdk

actions:
  opencode: opencode "$@"
```

This reference configuration shows that the SDK installs its skills into the workshop user profile and makes the `opencode` command available.

### 2. Start a workshop with this SDK

```bash
workshop launch
```

Open a shell in the workshop environment:

```bash
workshop shell
```

### 3. Prepare the workshop environment
To create a pull request at the end of the workflow, configure GitHub credentials inside the workshop environment:

```bash
sudo snap install gh --classic
gh auth login --scopes repo,workflow
```

### 4. Run the inference-snap flow

The SDK includes these skills:

- `inference-snap-structure`
- `github-workflows`
- `inference-snap-static-checks`
- `inference-snap-build-and-prompt-check`
- `inference-snap-create-pr`
- `inference-snap-pipeline`

To create a snap, run the pipeline skill, which orchestrates the full workflow. Open the OpenCode TUI inside the workshop environment with:

```bash
opencode
```

In the OpenCode TUI, run:

```
apply inference-snaps-sdk/agent-instructions.md
```

The agent requests the required inputs, executes the full workflow, and provides a final report with results and a pull request for the specified GitHub repository URL.

### 5. Connect OpenCode to an inference snap

OpenCode can be configured to connect directly to an inference snap API.

Create an `opencode.json` file with the following content:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "<MODELNAME>",
  "provider": {
    "inference-snap": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Local Inference Snap",
      "options": {
        "baseURL": "http://localhost:<PORT>/v1",
        "apiKey": "dummy"
      },
      "models": {
        "<MODELNAME>": {
          "name": "<MODELNAME> (local snap)"
        }
      }
    }
  }
}
```

The workshop must expose the snap's API port through the `opencode` plug and the `system` slot. If you are adapting this setup for another workshop, keep the API port consistent in both places:

```yaml
# workshop.yaml
name: inference-snap-dev
base: ubuntu@24.04
sdks:
  # Useful for testing the SDK in isolation, but not required to run the skills
  - name: vscode-remote
  - name: opencode
    channel: latest/stable
    plugs:
      api:
        interface: tunnel
        endpoint: localhost:<PORT>
  # The SDK should be cloned inside the workshop directory and built with `sdkcraft try` before launching the workshop
  - name: try-inference-snaps-sdk
  - name: system
    slots:
      api:
        interface: tunnel
        endpoint: localhost:<PORT>

actions:
  opencode: opencode "$@"
```

Connect the workshop plug to the slot:

```bash
workshop connect dev/opencode:api dev/system:api
```

The `opencode` CLI can now send requests directly to the inference snap API.
In the OpenCode TUI, run `/connect`, then select `Local Inference Snap` in the wizard. If prompted, use a dummy API key and select the desired model.

## License and copyright

TODO