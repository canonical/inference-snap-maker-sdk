# Inference Snaps SDK for Workshop

This SDK provides inference-snap authoring and validation skills for Workshop users, plus the OpenCode CLI in the workshop environment. It ships specialized skills for snap structure scaffolding, static validation, build and prompt verification, GitHub workflow generation, and PR creation, along with a pipeline skill that orchestrates these stages in sequence.

## Reference workshop

```yaml
# workshop.yaml
name: inference-snap-dev
base: ubuntu@24.04
sdks:
  # Useful for testing the SDK in isolation, but not required to run the skills
  - name: vscode-remote
  # The sdk is supposed to be cloned inside workshop directory and built with `sdkcraft try` before launching the workshop
  - name: try-inference-snaps-sdk
    plugs:
      api:
        interface: tunnel
        endpoint: localhost:<PORT>
  - name: system
    slots:
      api:
        interface: tunnel
        endpoint: localhost:<PORT>
```

This reference shows that the SDK installs its skills into the workshop user profile and makes `opencode` available.

## Using the SDK

### 1. Start a workshop with this SDK

```bash
workshop launch
```

### 2. Prepare Workshop environment
Once inside the workshop, run:

```bash
sudo snap install snapcraft --classic
```
This will allow you to build snaps with the `snapcraft` command.
Finally, in order to open the PR at the end of the flow, you need to have git credentials configured in the workshop environment. You can set them with:

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
- `inference-snap-from-example` (compatibility dispatcher)

In order to create a snap, you can run the pipeline skill which orchestrates the entire flow:

```bash
workshop shell
opencode
```
Inside the OpenCode TUI, run:
```bash
apply inference-snap-sdk/agent-instructions.md
```
The agent will ask for the required inputs and then execute the entire flow, providing a final report with results and a PR on the provided GitHub repository URL.


## License and copyright

TODO