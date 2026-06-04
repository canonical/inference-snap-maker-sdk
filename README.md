# Inference Snaps SDK for Workshop

This SDK provides inference snap authoring and validation skills for Workshop environments, together with the OpenCode CLI. It includes specialized skills for snap structure scaffolding, static validation, build and prompt verification, GitHub workflow generation, and pull request creation. It also includes a pipeline skill that orchestrates these stages in sequence.

## Reference workshop

```yaml
# workshop.yaml
name: inference-snap-dev
base: ubuntu@24.04
sdks:
  # Useful for testing the SDK in isolation, but not required to run the skills
  - name: vscode-remote
  # The SDK should be cloned inside the workshop directory and built with `sdkcraft try` before launching the workshop
  - name: opencode
    channel: latest/stable
    plugs:
      api:
        interface: tunnel
        endpoint: localhost:<PORT>
  - name: try-inference-snaps-sdk
  - name: system
    slots:
      api:
        interface: tunnel
        endpoint: localhost:<PORT>

actions:
  opencode: opencode "$@"
```

This reference configuration shows that the SDK installs its skills into the workshop user profile and makes the `opencode` command available.

## Using the SDK

### 1. Start a workshop with this SDK

```bash
workshop launch
```

### 2. Prepare Workshop environment
After entering the workshop, run:

```bash
sudo snap install snapcraft --classic
```

This enables snap builds with the `snapcraft` command.

To create a pull request at the end of the workflow, GitHub credentials must be configured in the workshop environment. Configure them with:

```bash
sudo snap install gh --classic
gh auth login --scopes repo,workflow
```

### 3. Run the inference-snap flow

The SDK includes these skills:

- `inference-snap-structure`
- `github-workflows`
- `inference-snap-static-checks`
- `inference-snap-build-and-prompt-check`
- `inference-snap-create-pr`
- `inference-snap-pipeline`

To create a snap, run the pipeline skill, which orchestrates the full workflow:

```bash
workshop shell
opencode
```

In the OpenCode TUI, run:

```bash
apply inference-snap-sdk/agent-instructions.md
```

The agent requests the required inputs, executes the full workflow, and provides a final report with results and a pull request for the specified GitHub repository URL.

### 4. Connect OpenCode to an inference snap

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

Connect the workshop plug to the slot:

```bash
workshop connect dev/opencode:api dev/system:api
```

The `opencode` CLI can now send requests directly to the inference snap API:

```bash
cd /PATH/TO/WORKSHOP
workshop refresh
workshop shell
opencode
```

In the OpenCode TUI, run `/connect`, then select `Local Inference Snap` in the wizard. If prompted, use a dummy API key and select the desired model.

## License and copyright

TODO