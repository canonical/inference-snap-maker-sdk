---
name: inference-snap-static-checks
description: Perform static quality checks for an inference snap (snapcraft.yaml consistency, component completeness, model signature verification).
trigger: Keywords like "check snapcraft yaml", "validate components", "empty components", "verify model signature", "static checks"
scope: user
---

# Inference Snap Static Checks

## Purpose

Validate correctness before build/run by checking metadata integrity, component completeness, and model authenticity signals.

## Checklist

1. snap/snapcraft.yaml checks:
   - Name/summary/description match target model.
   - Components declared under both parts and components sections.
   - App commands match existing binaries/symlinks.
   - Hooks and scripts referenced by snapcraft actually exist.
   - Engine/component naming consistency.
   - Each component name must be all lowercase, and must not contain any underscores or special characters other than hyphens. This is to ensure compatibility with snapcraft's naming conventions and to avoid potential issues during the build process.
   - The organize step in parts should be moving files from a correct path into a correct component name (or path). The component name is correct if it matches the name of the component declared in the components section, and the path is correct if it points to the expected location of the component's files after the build process.
   - Each directory under the `components` folder should have a matching component declared in the snapcraft.yaml, and each component declared in the snapcraft.yaml should have a corresponding directory under `components`. This ensures that all components are properly defined and organized within the snap structure.
   - If sharded models are included in the repository, each shard must be declared as a separate component in the snapcraft.yaml, and the organize step should correctly handle the placement of these shards into the appropriate component directories. This is important to ensure that all parts of the model are included in the snap and can be accessed correctly during runtime.
   - The `server` app adds `chat` the the main app's `ADDITIONAL_FEATURES`, while the `server-webui` app adds `webui` to the main app's `ADDITIONAL_FEATURES`. This is important for ensuring that the correct features are detected by the cli.
   - Each app should have the right snapcraft interfaces declared, for example servers should have `network-bind`.
2. Empty/incomplete component checks:
   - Each component directory has component.yaml.
   - Required artifacts exist (for model components: expected GGUF shards/files).
   - No placeholder-only component directories unless intentionally declared.
3. Engine sanity checks:
   - engine.yaml exists for each engine directory.
   - components list refers to existing components.
   - memory/disk constraints are realistic.
4. Model signature/provenance checks:
   - Verify source model URL and repository owner are the expected target.
   - Capture model filename, linked size, and ETag/hash headers when available.
   - If available, compare checksums of downloaded model file to trusted source metadata.
5. General consistency checks:
   - No duplicate component names across parts and components sections.
   - The install hooks sets ports to the requested values and sets the host to the default value of `127.0.0.1`
   - The `Makefile` downloads the right model files to the expected location under the `components` directory, and it should be called by the agent before packing the snap. If the model is sharded, all shards should land in the same directory.
   - Model files (like `*.gguf`) should be git ignored and not pushed with git-lfs.
6. component.yaml consistency checks:
   - Engine related components, like `llamacpp` or `llamacpp-cuda` specify their endpoints in component.yaml, along with the required environment variables, while the `server` script must have the execution flag set, and it should get the configuration from `modelctl` and runs the server with the correct parameters.
   - Model files components must have a `component.yaml` that specifies the MODEL_NAME with a value matching the actual model name and model path pointing to the correct location of the model files. If the model is sharded, `component.yaml` should also specify symlink creation to make all shards available under a common path.
   - MMPROJ related components must have a `component.yaml` that specifies the `MMPROJ_FILE` environment variable pointing to the correct location of the weights file. MODEL_NAME should not be set here.
7. Engines consistency checks:
   - Each engine directory must contain an `engine.yaml` that lists all components related to that engine, targeting the correct model variant and mmproj (if applicable).
   - The `engine.yaml` should specify realistic memory and disk constraints based on the expected resource usage of the engine and its components.
   - The components listed in `engine.yaml` must exist and be properly defined with their own `component.yaml`.
   - The `engine.yaml` should not reference components that are not declared in the snapcraft.yaml or that do not exist in the expected directory structure.
   - The `server` file must have the executable flag set
   - The engine name should contain a reference to the model size, if multiple sizes are packed inside a single snap.
   - The model description should reference the supported silicon and model variant.
   - Each `engine.yaml` file should specify a list of compatible devices.

## Output

- Findings ordered by severity (blocking/non-blocking).
- Exact file-level fixes required.
- Explicit pass/fail for: snapcraft schema consistency, component completeness, model signature checks.

## Rules

- Treat missing model artifact or mismatched model identity as blocking.
- Treat empty components as blocking unless explicitly intended.
- Do not mark validation passed if any blocking issue remains.
