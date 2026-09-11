# Inference Snap Structure RULESET

This ruleset is self-sufficient. Given a user request an agent MUST be able to scaffold a Canonical inference snap without external examples.

Terminology: **MUST** = required for build/runtime correctness; **SHOULD** =
recommended convention; **MAY** = optional.

---

## 1. Variant selector (read first)

Choose ONE packaging variant for each model artifact. Makefile download rules determine whether the model is a single file or split into multiple parts. The variant choice affects the snap structure and component layout.

- **Variant A - single component model.**
  One model component contains all model files.
  Use when the component payload is comfortably below Store limits.

- **Variant B - split model.**
  Model is split across multiple components because a single component would be
  too large (soft threshold around 5 GB in practice).
  Typical examples:
  - GGUF parts `...-00001-of-00004.gguf` etc.
  - OpenVINO/IR split across `...-1-of-2`, `...-2-of-2` components.

Both variants share most structure. Differences are primarily in:
- `parts.local-component-files` organization
- top-level `components:` entries
- `models/*/model.yaml` `layout:` flattening

---

## 2. Directory tree (annotated)

```text
<repo-root>/
  snap/
    snapcraft.yaml                       # MUST
    gui/
      icon-256.png                       # MAY (referenced by top-level `icon:`)
    hooks/
      install                            # MUST, executable
      post-refresh                       # MUST, executable
  engines/
    <engine-name>/
      engine.yaml                        # MUST
      server                             # MUST, executable
  models/
    <model-id>/
      model.yaml                         # MUST
  runtimes/
    <runtime-name>/
      runtime.yaml                       # MUST
  components/
    <component-name>/                    # MUST, >=1 model and >=1 runtime component
      <files...>                         # model/runtime payloads
  scripts/
    server.sh                            # MUST
    server-webui.sh                      # MUST
    completion.bash                      # MUST
  Makefile                               # SHOULD (used to download models)
  README.md                              # SHOULD
  LICENSE                                # SHOULD (empty file)
  LICENSE-<snap-name>                    # SHOULD (empty file)
  NOTICE                                 # SHOULD (legal attribution)
  .gitignore                             # SHOULD include: *.snap *.comp parts/ prime/ stage/ *.gguf components/ .craft/ .snapd-relocate/
  .gitmodules                            # MAY (for the `dev/` submodule)
  .gitattributes                         # SHOULD if using Git LFS: components/model*/*.gguf filter=lfs diff=lfs merge=lfs -text
  renovate.json                          # MAY
```

`server` files (in `engines/<name>/` and `components/<runtime>/`) MUST
have executable permission committed (`chmod +x`).

---

## 3. snap/snapcraft.yaml shape

### 3.1 Top-level fields

`{{SNAP_TITLE}}` is the friendly display name from the README `snap-title` frontmatter.
The `summary`/`description` body MUST mirror the completed template README:
one bullet per shipped engine, and `**Run:**` uses the bare command (no `--help`).

```yaml
name: {{SNAP_NAME}}
title: {{SNAP_TITLE}}
summary: Local AI with {{SNAP_TITLE}} inference snap
description: |
  {{MODEL_DESCRIPTION}}

  Use this snap to quickly install an optimized environment for local inference with {{SNAP_TITLE}}.

  The snap includes the following hardware-optimized inference engines:

  * cpu: Optimized for x64 and ARM (armv8, armv9) CPUs
  * nvidia-gpu: CUDA-enabled GPU acceleration
  # one bullet per engine the snap actually ships, matching the README

  The most suitable engine is automatically selected based on the available hardware.

  **Install:**

  `sudo snap install {{SNAP_NAME}}`

  **Run:**

  `{{SNAP_NAME}}`

  Some accelerators require extra drivers to be usable with this snap:
  https://documentation.ubuntu.com/inference-snaps/how-to/setup/drivers/

  **License:**

  The {{SNAP_TITLE}} model is provided by {{MODEL_VENDOR}} under the {{MODEL_LICENSE_NAME}} license:
  {{MODEL_LICENSE_URL}}

  The licenses of all the bundled software can be found after installing the snap at
  `/snap/{{SNAP_NAME}}/current/usr/share/doc`.

icon: snap/gui/icon-256.png   # only include when the icon file exists; omit otherwise or the build fails
website: https://documentation.ubuntu.com/inference-snaps
source-code: {{REMOTE_REPO_URL}}
issues: https://github.com/canonical/inference-snaps/issues

adopt-info: version

base: core24
grade: stable
confinement: strict
compression: lzo

assumes:
  - snapd2.68

platforms:
  amd64:
  arm64:
```

### 3.2 Environment

MUST include:

```yaml
environment:
  SNAP_COMPONENTS: /snap/$SNAP_INSTANCE_NAME/components/$SNAP_REVISION
  ARCH_TRIPLET: $CRAFT_ARCH_TRIPLET_BUILD_FOR
```

MAY include OpenCL env when Intel/OpenCL runtime is used:

```yaml
  OCL_ICD_VENDORS: $SNAP/etc/OpenCL/vendors
```

### 3.3 Plugs and slots

MUST declare home plug for sideloading:

```yaml
plugs:
  # To allow sideloading models by root
  home:
    read: all
```

OpenCL/Intel runtimes MAY additionally declare layout binds for ICD and library
paths.

### 3.5 Hooks declaration

```yaml
hooks:
  install:
    plugs: &install-plugs
      - hardware-observe
      - opengl
  post-refresh:
    plugs: *install-plugs
```

### 3.6 Parts - common

MUST include these functional groups:
- `version`
```yaml
version:
  after: [cli]
  plugin: nil
  source: . # To regenerate the git hash on every change
  build-packages:
    - jq
    - git
  override-pull: |
    # Do nothing. This significantly reduces build time when sourcing large files.
  override-build: |
    set -eo pipefail
    cli_version=$($CRAFT_STAGE/bin/modelctl version --format=json | jq .cli --raw-output)
    git_hash=$(git -C "$SNAPCRAFT_PROJECT_DIR" describe --always)
    craftctl set version="${cli_version#v}+${git_hash}"
```
- `cli and cli-dependencies`
```yaml
cli:
    source:
      - on amd64: https://github.com/canonical/inference-snaps-cli/releases/download/{{LAST_RELEASE_TAG}}/inference-snaps-cli-linux-amd64.tar.xz
      - on arm64: https://github.com/canonical/inference-snaps-cli/releases/download/{{LAST_RELEASE_TAG}}/inference-snaps-cli-linux-arm64.tar.xz
    plugin: dump
    override-build: |

      # For tab completion
      ln --symbolic ./modelctl bin/{{SNAP_NAME}}

      craftctl default
    
  cli-dependencies:
    plugin: nil
    stage-packages:
      - pciutils # lspci
```
- `webui` and`engines` and `models` and `runtimes` and `scripts`:
```yaml
webui:
  plugin: dump
  source: https://github.com/canonical/inference-snaps-webui/releases/download/v1.1.0/inference-snaps-webui.tar.xz
  organize:
    "*": webui/

engines:
  source: engines
  plugin: dump
  organize:
    "*": engines/

models:
  source: models
  plugin: dump
  organize:
    "*": models/

runtimes:
  source: runtimes
  plugin: dump
  organize:
    "*": runtimes/

scripts:
  source: scripts
  plugin: dump
  stage-packages:
    - jq
  organize:
    "server.sh": bin/
    "server-webui.sh": bin/
```
- `local-component-files` SHOULD use `plugin: cmake` with an `override-build` copy workaround to avoid unwanted `dump` behavior for very large payloads, and MUST end with `prime: [-*]` so unorganized component sources do not leak into the base snap. ADDED — canonical shape:
```yaml
  local-component-files:
    plugin: cmake
    source: components
    override-build: |
      cp -rf --archive --link --no-dereference ${CRAFT_PART_SRC}/* ${CRAFT_PART_INSTALL}
    organize:
      "model-<slug>/*": (component/model-<slug>)
      "mmproj-<slug>/*": (component/mmproj-<slug>)
      # one line per part for split models, mapping each part file to its component
    prime:
      - -*   # exclude everything not explicitly organized
```
- runtime payload parts, here is an example for `llamacpp`, eventually use and adapt it also for other runtimes (for example `llamacpp-cuda`, `llamacpp-rocm`, `openvino-model-server`)
```yaml
llamacpp:
    plugin: dump
    source:
      - on amd64: https://github.com/canonical/llama.cpp-builds/releases/download/b9611/llamacpp-amd64.tar.gz
      - on arm64: https://github.com/canonical/llama.cpp-builds/releases/download/b9611/llamacpp-arm64.tar.gz
    override-build: |
      # Move license files to be included in the snap, not the component
      mkdir -p $CRAFT_PRIME/usr/share/doc/
      mv licenses $CRAFT_PRIME/usr/share/doc/llama.cpp

      craftctl default
    stage-packages:
      - libgomp1
    organize:
      # move everything, including the staged packages
      "*": (component/llamacpp)
```
- notice
```yaml
  notice:
    plugin: nil
    source: NOTICE
    source-type: file
    override-build: |
      license_dir=$CRAFT_PART_INSTALL/usr/share/doc
      mkdir -p $license_dir
      cp NOTICE $license_dir/
      cp $SNAPCRAFT_PROJECT_DIR/LICENSE $license_dir/
      cp $SNAPCRAFT_PROJECT_DIR/LICENSE-{{SNAP_NAME}} $license_dir/
```
NVIDIA detection SHOULD use a minimal `cli-nvidia-smi` part that copies only
`/usr/bin/nvidia-smi` and related license text.

**Runtime & CLI artifact sources** (pin a known-good tag; verify each URL
returns HTTP 200 before use). A literal scaffold with placeholder parts will NOT
build — use these real sources:

- CLI: `https://github.com/canonical/inference-snaps-cli/releases/download/<CLI_TAG>/inference-snaps-cli-linux-{amd64,arm64}.tar.xz`
- WebUI: `https://github.com/canonical/inference-snaps-webui/releases/download/<WEBUI_TAG>/inference-snaps-webui.tar.xz`, organized into `webui/`.
- llama.cpp runtimes: `https://github.com/canonical/llama.cpp-builds/releases/download/<LLAMA_BUILD>/llamacpp-{amd64,arm64}[+cuda12.9|+rocm|+onemkl].tar.gz`,

The same `<CLI_TAG>` MUST be used by the `cli` part AND by the
`pr-checks` CI job's checkout `ref`.

### 3.7 Components (top-level)

Every component payload under `components/` MUST have a matching top-level
`components:` entry in `snapcraft.yaml`.

All component entries MUST be `type: standard`.

For split models, use YAML anchors to avoid duplication:

```yaml
components:
  model-{{MODEL_SLUG}}-1-of-{{N_PARTS}}: &model
    type: standard
    summary: ...
    description: ...
  model-{{MODEL_SLUG}}-2-of-{{N_PARTS}}:
    <<: *model
```

### 3.8 Apps

MUST define three apps:

```yaml
apps:
  {{SNAP_NAME}}:
    command: bin/modelctl
    completer: bin/snap-completer.bash
    plugs:
      - hardware-observe
      - opengl
      - network
      - desktop # To open the webui in the browser
    environment:
      ADDITIONAL_FEATURES: chat, webui

  server:
    command: bin/server.sh
    daemon: simple
    plugs:
      - network-bind
      - hardware-observe
      - opengl
      - home
      # MAY include process-control only if a llama.cpp-rocm is present

  server-webui:
    command: bin/server-webui.sh
    daemon: simple
    plugs:
      - network-bind
```

---

## 4. Hook conventions

### 4.1 snap/hooks/install

MUST:
1. Use `#!/bin/bash -eu`.
2. Log to syslog using tagged stdout/stderr redirection.
3. Seed package config values:
   - `http.port={{PORT}}`
   - `http.host=127.0.0.1`
   - `webui.http.port={{WEBUI_PORT}}`
   - `webui.http.host=127.0.0.1`
   - `verbose=false`
4. Auto-select engine with non-interactive fallback:

```bash
modelctl use-engine --auto --components --assume-yes --fallback=cpu
```
Example:

```bash
#!/bin/bash -eu

tag="snap.$SNAP_INSTANCE_NAME.hook.install"

# Redirect stdout to stdout+syslog
exec 1> >(tee >(logger --tag=$tag))
# Redirect stderr to stderr+syslog
exec 2> >(logger --stderr --priority error --tag=$tag)

#
# Set generic config
#

modelctl set --package http.port="8352"
modelctl set --package http.host="127.0.0.1"
modelctl set --package verbose="false"

# Webui server config
modelctl set --package webui.http.port="8353"
modelctl set --package webui.http.host="127.0.0.1"

#
# Auto select an engine
#

modelctl use-engine --auto --components --assume-yes --fallback=cpu
```

### 4.2 snap/hooks/post-refresh

MUST:
1. Use same syslog tagging pattern.
2. Refresh active engine:

```bash
modelctl use-engine --fix --assume-yes --fallback=cpu
```
Example:

```bash
#!/bin/bash -eu

tag="snap.$SNAP_INSTANCE_NAME.hook.post-refresh"

# Redirect stdout to stdout+syslog
exec 1> >(tee >(logger --tag=$tag))
# Redirect stderr to stderr+syslog
exec 2> >(logger --stderr --priority error --tag=$tag)

#
# Refresh the active engine
#
modelctl use-engine --fix --assume-yes --fallback=cpu
```
---

## 5. Script conventions

### 5.1 scripts/server.sh

MUST select active engine and execute its server script:

```bash
#!/bin/bash
set -euo pipefail
engine="$(modelctl status --wait-for-components --format=json | jq -r .engine)"
exec modelctl run -- "$SNAP/engines/$engine/server" "$@"
```

### 5.2 scripts/server-webui.sh

MUST obtain host/port from modelctl and run webui service:

```bash
#!/bin/bash
set -euo pipefail

port="$(modelctl get webui.http.port)"
host="$(modelctl get webui.http.host)"

exec modelctl serve-webui "$SNAP/webui" --port "$port" --host "$host"

```

---

## 6. Engine schema (engines/<name>/engine.yaml)

### 6.1 Required keys

```yaml
name: {{ENGINE_NAME}}
summary: {{SHORT_SUMMARY}}
description: {{OPTIONAL_MULTILINE}}
vendor: Canonical Ltd
devices:
  anyof: ... and/or allof: ...
runtime: {{RUNTIME_NAME}}
model:
  default: {{MODEL_ID}}
  options:
    - {{MODEL_ID}}
    - ...
```

`configurations.sleep-idle-seconds: 600` SHOULD be set for llama.cpp-based
engines.

`experimental: true` MAY be set for non-default experimental engines.

**Multiple model sizes in one snap:** keep ONE engine per backend
(e.g. `cpu`, `nvidia-gpu`) and list every size in `model.options` with one as
`model.default`. Encode the size in the *model id* (e.g.
`4b-q4-k-xl-gguf`, `9b-q4-k-m-gguf`).

### 6.2 Devices

Allowed patterns include:
- CPU-only (`type: cpu`, architecture predicates)
- NVIDIA (`vendor-id: 0x10de` + architecture constraints)
- AMD (`vendor-id: 0x1002` + microarchitecture list)
- Intel CPU (`manufacturer-id: GenuineIntel`)
- Intel GPU (`vendor-id: 0x8086`, often with `device-id` and `vram` minimum)

### 6.3 Engine server scripts

For llama.cpp runtimes, canonical server shape:

```bash
#!/bin/bash -eu

port="$(modelctl get http.port)"
host="$(modelctl get http.host)"
sleep_idle_seconds="$(modelctl get sleep-idle-seconds)"

extra_args=()

verbose="$(modelctl get verbose)"
if [ "${verbose}" = "true" ]; then
  extra_args+=(--verbose)
fi

mmproj_args=()
if [ -n "${MMPROJ_FILE:-}" ]; then
  mmproj_args=(--mmproj "$MMPROJ_FILE")
fi

set -x
# Adding --no-warmup to skip the model warmup phase during server startup in order to reduce resource usage if not needed.
exec llama-server \
  --model "$MODEL_FILE" \
  --alias "$MODEL_NAME" \
  "${mmproj_args[@]}" \
  --port "$port" \
  --host "$host" \
  --no-warmup \
  --sleep-idle-seconds "$sleep_idle_seconds" \
  "${extra_args[@]}" \
  "$@"
```

For OpenVINO runtimes, server script typically runs `ovms` using `MODEL_PATH`
and `MODEL_NAME`.
```bash
#!/bin/bash -eu

port="$(modelctl get http.port)"
host="$(modelctl get http.host)"

extra_args=()

verbose="$(modelctl get verbose)"
if [ "${verbose}" = "true" ]; then
  extra_args+=(--log_level DEBUG)
fi

# Set --pipeline_type VLM to work around an issue with the default
# continuous batching pipeline, VLM_CB

set -x
ovms \
    --rest_port "$port" \
    --rest_bind_address "$host" \
    --model_name "$MODEL_NAME" \
    --model_path "$MODEL_PATH" \
    --pipeline_type VLM \
    --task text_generation \
    --target_device GPU \
    "${extra_args[@]}" \
    "$@"
```

---

## 7. Runtime schema (runtimes/<name>/runtime.yaml)

MUST define runtime executable environment and exposed server API metadata:

```yaml
name: {{RUNTIME_NAME}}
servers:
  openai:
    protocol: http
    base-path: /v1
environment:
  - PATH=$PATH:$SNAP_COMPONENTS/{{RUNTIME_COMPONENT}}/bin
  - LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$SNAP_COMPONENTS/{{RUNTIME_COMPONENT}}/lib
  - LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$SNAP_COMPONENTS/{{RUNTIME_COMPONENT}}/usr/lib/$ARCH_TRIPLET
components:
  - {{RUNTIME_COMPONENT}}
```

Non-llama runtimes MAY declare additional server protocols (for example OVMS
`tensorflow-serving` `/v1`, `kserve` `/v2`, `openai` `/v3`).
For example, OpenVINO OVMS runtime may declare multiple protocols:

```yaml
servers:
  tensorflow-serving:
    protocol: http
    base-path: /v1
  kserve:
    protocol: http
    base-path: /v2
  openai:
    protocol: http
    base-path: /v3

environment:
  # Add OVMS binaries
  - PATH=$PATH:$SNAP_COMPONENTS/openvino-model-server/bin
  # Add staged shared objects
  - LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$SNAP_COMPONENTS/openvino-model-server/lib
  - LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$SNAP_COMPONENTS/openvino-model-server/usr/lib/$ARCH_TRIPLET
  - LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$SNAP_COMPONENTS/openvino-model-server/usr/local/lib
  - LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$SNAP/usr/local/lib

components:
  - openvino-model-server
```

---

## 8. Model schema (models/<id>/model.yaml)

### 8.1 Required keys

```yaml
id: {{MODEL_ID}} # something like 4b-q4-k-xl-gguf or 4b-q4-k-xl-ov
name: {{MODEL_FAMILY_OR_SIZE}} #same as id
description: {{HUMAN_DESCRIPTION}}
model-card-url: {{URL}}
quantization: {{QUANT_LABEL}}
disk-size: {{SIZE}}   # integer + binary unit only, e.g. 3420M / 6300M / 16163M.
                      # Decimals or a trailing "B" (e.g. "3.4GB") break `modelctl list-models`
                      # with: strconv.ParseUint: parsing "3.4GB": invalid syntax
capabilities:
  - text
components:
  - {{MODEL_COMPONENT_1}}
environment:
  - MODEL_NAME={{MODEL_ALIAS}}
```

`capabilities` SHOULD include applicable values from:
`text`, `vision`, `thinking`, `tools`, `audio`.

### 8.2 Environment conventions

For GGUF single-file model:

```yaml
environment:
  - MODEL_FILE=$SNAP_COMPONENTS/{{MODEL_COMPONENT}}/{{MODEL_FILE}}
  - MODEL_NAME={{MODEL_ALIAS}}
  - MMPROJ_FILE=$SNAP_COMPONENTS/{{MMPROJ_COMPONENT}}/{{MMPROJ_FILE}}   # if multimodal
```

For split models, set an intermediate directory and flatten with layout:

```yaml
environment:
  - MODEL_PARTS_DIR=/tmp/{{MODEL_SLUG}}-parts
  - MODEL_FILE=$MODEL_PARTS_DIR/{{PART_1_FILE}}
  - MODEL_NAME={{MODEL_ALIAS}}
layout:
  $MODEL_PARTS_DIR/{{PART_1_FILE}}:
    symlink: $SNAP_COMPONENTS/{{COMPONENT_1}}/{{PART_1_FILE}}
  ...
```

OpenVINO split models MAY use `MODEL_DIR` / `MODEL_PATH` plus explicit layout
for each required file.

---

## 9. Naming agreement checks (blocking)

The following names MUST agree exactly:

- `apps.{{SNAP_NAME}}.command` points to existing `bin/{{SNAP_NAME}}`.
- Every `engine.yaml runtime` exists as `runtimes/<runtime>/runtime.yaml`.
- Every `engine.yaml model.default/options` entry exists as `models/<id>/model.yaml`.
- Every component listed in `models/*/model.yaml#components` exists in top-level
  `snapcraft.yaml#components` and is materialized by parts into
  `(component/<name>)`.
- `MODEL_NAME` in model environment matches expected API model identifier.
- `MODEL_FILE` / `MMPROJ_FILE` / `MODEL_PATH` environment values resolve to
  files or directories that exist at runtime.

---

## 10. Common blocking failures

| Failure | Preventing rule |
| --- | --- |
| Engine references unknown model id | Section 9 model id agreement |
| Engine references unknown runtime | Section 9 runtime agreement |
| Model component missing from top-level components | Sections 3.7 + 9 |
| Split model cannot load because files live in separate components | Section 8.2 layout flattening |
| Wrong model id in `/v1/models` | `MODEL_NAME` plus `--alias` in llama server |
| Auto-selection breaks install path | Section 4 with `--fallback=cpu` |
| WebUI not reachable | ports seeded in install + `network-bind` on `server-webui` |
---

## 11. Deterministic scaffolding algorithm

Given:
- model ids and artifacts
- target ports
- capabilities
- desired backends (cpu/nvidia/amd/intel/openvino)
- component size constraints

1. Create skeleton (`engines/`, `models/`, `runtimes/`, `components/`, `scripts/`, `snap/`).
2. Generate `snap/snapcraft.yaml` with required parts/apps/components.
3. Generate hooks with config seeding and engine select/fix commands.
4. Generate each runtime descriptor (`runtime.yaml`).
5. Generate each model descriptor (`model.yaml`), including layout for split models.
6. Generate each engine descriptor and matching engine `server` script.
7. Verify naming agreements from section 9.
8.  Run static checks and fail on any mismatch.

---

## 12. Substitution table

| Substitution | Meaning | Example |
| --- | --- | --- |
| `{{SNAP_NAME}}` | snap/app name | `gemma4`, `fastcontext-1-0` |
| `{{MODEL_ID}}` | `models/<id>/` identifier | `e4b-q4-k-m-gguf` |
| `{{MODEL_ALIAS}}` | runtime model id (API visible) | `e4b-q4-k-m` |
| `{{MODEL_FILE}}` | model file basename | `gemma-4-E4B-it-Q4_K_M.gguf` |
| `{{MMPROJ_FILE}}` | mmproj basename | `mmproj-gemma-4-E4B-it-Q8_0.gguf` |
| `{{RUNTIME_NAME}}` | runtime descriptor name | `llamacpp`, `openvino-model-server` |
| `{{RUNTIME_COMPONENT}}` | runtime component payload | `llamacpp`, `openvino-model-server` |
| `{{ENGINE_NAME}}` | engine directory and `name` | `cpu`, `nvidia-gpu`, `intel-gpu` |
| `{{PORT}}` | inference HTTP port | `8336` |
| `{{WEBUI_PORT}}` | webui HTTP port | `8337` |
| `{{N_MODEL_PARTS}}` | number of split artifacts | `4` |

