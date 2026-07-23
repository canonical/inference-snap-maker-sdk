# Inference Snaps SDK for Workshop

This SDK provides inference snap authoring and validation skills for Workshop environments, together with the OpenCode CLI.

## Skills
The SDK includes the following skills:
- `inference-snap-pipeline` — orchestrates the below skills in a sequential
- `inference-snap-structure` — scaffolds the snap structure and files
- `github-workflows` — creates GitHub workflows for CI/CD
- `inference-snap-static-checks` — performs static checks on the snap structure and files
- `inference-snap-build-and-test` — builds the snap and runs smoke tests

## Agents
The SDK includes the following specialized agents, each of them defines input and output for a specific stage of the inference snap workflow:
- `inference-snap-structure-stage` — coupled with the `inference-snap-structure` skill
- `inference-snap-github-workflows-stage` — coupled with the `github-workflows` skill
- `inference-snap-static-checks-stage` — coupled with the `inference-snap-static-checks` skill
- `inference-snap-build-and-test-stage` — coupled with the `inference-snap-build-and-test` skill

Both agents and skills are installed in /home/workshop/.agents directory by the hook `setup-project` and are available in the OpenCode TUI.

## How to use the SDK in a Workshop to pack a new snap

### 1. Prepare the workshop environment

- Use the `inference-snaps-template` [repository](https://github.com/canonical/inference-snap-template) to create the new inference snap Github repository.
- Modify the Makefile to download the desired models.
- Modify the README.md by compiling the required inputs listed at the top of the file between the `<!--` and `-->` comments.

### 3. Start a workshop with this SDK

```bash
workshop launch
```

Open a shell in the workshop environment:

```bash
workshop shell
```

### 4. Run the inference-snap flow
Open the OpenCode TUI inside the workshop environment with:

```bash
opencode --auto
```

In the OpenCode TUI, run:

```
start snap packaging pipeline
```

## License and copyright

TODO