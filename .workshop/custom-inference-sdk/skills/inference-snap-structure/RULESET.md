# Inference Snap Structure RULESET

This ruleset is self-sufficient. Given only a user request of the form
"snap up model X with HTTP port P and WebUI port Q" plus the substitution
table at the end of this file, a future agent MUST be able to scaffold a
Canonical inference snap without referring to any external example.

Terminology: **MUST** = required for the snap to build/run correctly;
**SHOULD** = matches established convention, deviate only with reason;
**MAY** = optional, model-specific.

---

## 1. Variant selector (read first)

Choose ONE of the two variants based on the on-disk size of the primary
quantized GGUF model file:

- **Variant A — single-file model.** The model GGUF is one file < 5 GB.
  Use this variant for all engines: CPU, NVIDIA, AMD/ROCm.
- **Variant B — sharded model.** The model is delivered as multiple
  numbered GGUF shards because the single artifact exceeds the Snap
  Store per-component soft limit (~5 GB). Trigger threshold: any single
  GGUF > 5 GB, or total weights > 5 GB.

Both variants share everything except: (a) `parts` for model files,
(b) `components` block model entries, (c) shard-1 `component.yaml`
declaring a `layout`. All other rules below apply to both unless
explicitly tagged "Variant A only" or "Variant B only".

---

## 2. Directory tree (annotated)

```
<repo-root>/
  snap/
    snapcraft.yaml                       # MUST
    hooks/
      install                            # MUST, executable
      post-refresh                       # MUST, executable
  engines/
    <engine-name>/                       # MUST, >=1 engine
      engine.yaml                        # MUST
      server                             # MUST, executable
  components/
    <runtime-component>/                 # MUST, >=1 (e.g. llamacpp)
      component.yaml                     # MUST
      server                             # MUST, executable
    <model-component>/                   # MUST, >=1 model component
      component.yaml                     # MUST
      <model>.gguf                       # MUST at build time (Git LFS ok)
    <mmproj-component>/                  # MAY (vision/multimodal only)
      component.yaml
      <mmproj>.gguf
  scripts/
    server.sh                            # MUST
    server-webui.sh                      # MUST
    completion.bash                      # MUST
    export-shared-configs.sh             # MAY (only if status/owui slots declared)
  download-models.sh                     # SHOULD (developer convenience)
  README.md                              # SHOULD
  NOTICE                                 # SHOULD (legal attribution)
  .gitignore                             # SHOULD include: *.snap *.comp parts/ prime/ stage/ *.gguf
  .gitmodules                            # MAY (for the `dev/` submodule)
  .gitattributes                         # SHOULD if using Git LFS: components/model*/*.gguf filter=lfs diff=lfs merge=lfs -text
  renovate.json                          # MAY
```

`server` files (in `engines/<name>/` and `components/<runtime>/`) MUST
have executable permission committed (`chmod +x`).

---

## 3. `snap/snapcraft.yaml` shape

### 3.1 Top-level fields (both variants)

```yaml
name: {{SNAP_NAME}}                       # MUST, lowercase, hyphenated; matches CLI alias
base: core24                              # MUST
summary: Inference Snap for {{MODEL_DISPLAY_NAME}}
description: See https://snapcraft.io/{{SNAP_NAME}}

adopt-info: version                       # MUST, version is computed by the `version` part

grade: stable                             # MUST be `stable` for store release; `devel` only for dev branches
confinement: strict                       # MUST
compression: lzo                          # MUST

assumes:                                  # SHOULD (required for components support)
  - snapd2.68

platforms:                                # SHOULD include amd64 and arm64
  amd64:
  arm64:
```

### 3.2 `environment` (top-level)

MUST set at minimum:

```yaml
environment:
  SNAP_COMPONENTS: /snap/$SNAP_INSTANCE_NAME/components/$SNAP_REVISION
  ARCH_TRIPLET: $CRAFT_ARCH_TRIPLET_BUILD_FOR
```

MAY add (only if matching `slots:` are declared — see 3.4):

```yaml
  STATUS_SHARE: *status-share
  OWUI_SHARE: *owui-share
```

MAY add (Nemotron-style, when the snap ships OpenCL ICDs or relies on them):

```yaml
  OCL_ICD_VENDORS: $SNAP/etc/OpenCL/vendors
```

### 3.3 `plugs` (top-level)

MUST declare for sideloading from user home:

```yaml
plugs:
  home:
    read: all
```

### 3.4 `slots` (optional, content-sharing)

MAY declare for inter-snap content sharing (e.g. exposing status to a
companion app, or feeding an Open WebUI snap). If declared, MUST be
paired with `export-shared-configs.sh` in `scripts/`.

```yaml
slots:
  status:
    interface: content
    source:
      read:
        - &status-share $SNAP_DATA/share/status
  open-webui:
    interface: content
    content: open-webui-config
    source:
      read:
        - &owui-share $SNAP_DATA/share/open-webui
```

### 3.5 `layout` (only if model needs host bind mounts)

MAY include the WSL bind workaround (helps NVIDIA users on WSL2). Pair
with the symlink-creation block in `hooks/install` and `hooks/post-refresh`.

```yaml
layout:
  /usr/lib/wsl:
    bind: $SNAP_COMMON/usr/lib/wsl
```

### 3.6 `hooks`

MUST be:

```yaml
hooks:
  install:
    plugs: &install-plugs
      - hardware-observe
      - opengl
  post-refresh:
    plugs: *install-plugs
```

### 3.7 `parts` — common parts (both variants)

The following parts MUST exist verbatim (substituting `{{SNAP_NAME}}`
and the CLI release tag as appropriate):

```yaml
parts:
  version:
    plugin: dump
    source: .
    build-packages:
      - git                               # use `git-lfs` if model files are LFS-tracked
    override-pull: |
      hash=$(git -C $SNAPCRAFT_PROJECT_DIR describe --always)
      craftctl set version="{{VERSION_PREFIX}}+$hash"

  cli:
    source:
      - on amd64: https://github.com/canonical/inference-snaps-cli/releases/download/{{CLI_TAG}}/inference-snaps-cli-linux-amd64.tar.xz
      - on arm64: https://github.com/canonical/inference-snaps-cli/releases/download/{{CLI_TAG}}/inference-snaps-cli-linux-arm64.tar.xz
    plugin: dump
    override-build: |
      mkdir -p bin
      mv cli bin/modelctl
      ln --symbolic ./modelctl bin/{{SNAP_NAME}}
      craftctl default
    stage-packages:
      - pciutils                          # provides lspci

  webui:
    plugin: dump
    source: https://github.com/canonical/inference-snaps-webui/releases/download/{{WEBUI_TAG}}/inference-snaps-webui.tar.xz
    organize:
      "*": webui/

  engines:
    source: engines
    plugin: dump
    organize:
      "*": engines/

  scripts:
    source: scripts
    plugin: dump
    stage-packages:
      - jq
    organize:
      "server.sh": bin/
      "server-webui.sh": bin/
      "completion.bash": bin/
      # Add when status/owui slots are declared:
      # "export-shared-configs.sh": bin/
```

NVIDIA detection MAY be done via a separate `cli-nvidia-smi` part (copies
`/usr/bin/nvidia-smi` only, leaving libraries to the `opengl` interface):

```yaml
  cli-nvidia-smi:
    plugin: nil
    build-packages:
      - nvidia-utils-580
    override-build: |
      mkdir -p $CRAFT_PART_INSTALL/usr/bin
      cp /usr/bin/nvidia-smi $CRAFT_PART_INSTALL/usr/bin/
      license_dir=$CRAFT_PART_INSTALL/usr/share/doc/nvidia-utils-580/
      mkdir -p $license_dir
      cp /usr/share/doc/nvidia-utils-580/copyright $license_dir/
```

Equivalent alternative: add `nvidia-utils-580` to the `cli` part's
`stage-packages`. Pick one — do NOT do both.

### 3.8 `parts` — runtime (llama.cpp) parts

For each runtime backend the snap supports, declare a part that drops
files into `(component/<name>)`:

```yaml
  llamacpp:
    plugin: dump
    source:
      - on amd64: https://github.com/canonical/llama.cpp-builds/releases/download/{{LLAMACPP_TAG}}/llamacpp-amd64.tar.gz
      - on arm64: https://github.com/canonical/llama.cpp-builds/releases/download/{{LLAMACPP_TAG}}/llamacpp-arm64.tar.gz
    stage-packages:
      - libgomp1
    organize:
      "licenses": usr/share/doc/llama.cpp
      "*": (component/llamacpp)

  llamacpp-cuda:
    plugin: dump
    source:
      - on amd64: https://github.com/canonical/llama.cpp-builds/releases/download/{{LLAMACPP_TAG}}/llamacpp-amd64+cuda12.tar.gz
      - on arm64: https://github.com/canonical/llama.cpp-builds/releases/download/{{LLAMACPP_TAG}}/llamacpp-arm64+cuda12.tar.gz
    stage-packages:
      - libgomp1
    organize:
      "licenses": usr/share/doc/llama.cpp+cuda
      "*": (component/llamacpp-cuda)

  llamacpp-rocm-amd64:                    # MAY (amd64 only)
    plugin: dump
    source: https://github.com/canonical/llama.cpp-builds/releases/download/{{LLAMACPP_TAG}}/llamacpp-amd64+rocm.tar.gz
    stage-packages:
      - libgomp1
      - libatomic1
    organize:
      "licenses": usr/share/doc/llama.cpp+rocm
      "*": (component/llamacpp-rocm)
    override-pull: &amd64-only |
      [ "$CRAFT_ARCH_BUILD_FOR" == "amd64" ] && craftctl default || exit 0
    override-build: *amd64-only
    override-stage: *amd64-only
    override-prime: *amd64-only
```

The component subdirectory ALSO needs local files (the `server` script
and `component.yaml`). Both variants handle this differently from the
weights — see 3.9 and the next section.

For Variant A, a single `local-component-files` part SHOULD organize
every component-local file into its component prime directory:

```yaml
  local-component-files:
    plugin: dump
    source: components
    organize:
      "llamacpp/*": (component/llamacpp)
      "llamacpp-cuda/*": (component/llamacpp-cuda)
      "llamacpp-rocm/*": (component/llamacpp-rocm)
      "model-{{MODEL_SLUG}}-gguf/*": (component/model-{{MODEL_SLUG}}-gguf)
      "mmproj-{{MMPROJ_SLUG}}-gguf/*": (component/mmproj-{{MMPROJ_SLUG}}-gguf)
    prime:
      - -*                                # exclude anything not explicitly organized
```

For Variant B, the runtime-component local files are usually inlined
into per-backend `*-local-files` parts:

```yaml
  llamacpp-local-files:
    plugin: dump
    source: components/llamacpp
    organize:
      "*": (component/llamacpp)

  llamacpp-cuda-local-files:
    plugin: dump
    source: components/llamacpp-cuda
    organize:
      "*": (component/llamacpp-cuda)
```

### 3.9 `parts` — model parts

**Variant A (single-file model).** Routed through `local-component-files`
above. Nothing else required. Each model component directory contains
exactly: `component.yaml` + one `.gguf` file (plus optional README).

**Variant B (sharded model).** MUST add a dedicated part that bypasses
the staging lifecycle, copying shards directly into per-shard component
prime directories. Two equivalent forms exist:

Form B1 (lifecycle-bypass, recommended for very large models — saves disk):

```yaml
  model-{{MODEL_SLUG}}:
    plugin: nil
    # source intentionally omitted to avoid copying shards through stage.
    override-prime: |
      set +x
      source_dir="$CRAFT_PROJECT_DIR/components/model-{{MODEL_SLUG}}"
      for i in {1..{{N_SHARDS}}}; do
        target_env="CRAFT_COMPONENT_MODEL_{{MODEL_SLUG_UPPER}}_${i}_OF_{{N_SHARDS}}_PRIME"
        target_comp="${!target_env}"
        cp -v "$source_dir/{{MODEL_FILE_PREFIX}}-0000${i}-of-{{N_SHARDS_PADDED}}.gguf" "$target_comp/"
        if [ "$i" -eq 1 ]; then
          for f in "$source_dir"/*; do
            if [ -f "$f" ]; then
              case "$f" in
                *.gguf) ;;
                *) cp -v "$f" "$target_comp/" ;;
              esac
            fi
          done
        else
          echo "# No config" > "$target_comp/component.yaml"
        fi
      done
```

Form B2 (explicit `organize` in `local-component-files` — works when
the build host can spare the disk):

```yaml
  local-component-files:
    plugin: dump
    source: components
    organize:
      "model-{{MODEL_SLUG}}/component.yaml": (component/model-{{MODEL_SLUG}}-1-of-{{N_SHARDS}})
      "model-{{MODEL_SLUG}}/{{FILE_PREFIX}}-00001-of-{{N_SHARDS_PADDED}}.gguf": (component/model-{{MODEL_SLUG}}-1-of-{{N_SHARDS}})
      "model-{{MODEL_SLUG}}/component-2-of-{{N_SHARDS}}.yaml": (component/model-{{MODEL_SLUG}}-2-of-{{N_SHARDS}})/component.yaml
      "model-{{MODEL_SLUG}}/{{FILE_PREFIX}}-00002-of-{{N_SHARDS_PADDED}}.gguf": (component/model-{{MODEL_SLUG}}-2-of-{{N_SHARDS}})
      # ...repeat for shards 3..N
    prime:
      - -*
```

For Form B2, `components/model-{{MODEL_SLUG}}/component-{i}-of-{N}.yaml`
files (for i >= 2) MUST exist on disk as placeholder comment files
("`# Used in shards different from the first shard`").

In both forms, the per-shard component prime directories MUST contain a
`component.yaml`. Only shard 1's `component.yaml` carries meaningful
content (environment + layout); shards 2..N carry a placeholder comment.

### 3.10 `apps`

MUST define three apps:

```yaml
apps:
  {{SNAP_NAME}}:                          # CLI alias (symlink to modelctl)
    command: bin/{{SNAP_NAME}}
    completer: bin/completion.bash
    plugs:
      - hardware-observe
      - opengl
      - network
      - desktop                           # used to open the webui in a browser
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
      # MAY also include: process-control, network

  server-webui:
    command: bin/server-webui.sh
    daemon: simple
    plugs:
      - network-bind
    # MAY also include:
    # environment:
    #   UI_ASSETS_DIRECTORY: $SNAP/webui
```

The CLI app name MUST equal `{{SNAP_NAME}}` so the `bin/{{SNAP_NAME}}`
symlink (created by the `cli` part) resolves.

### 3.11 `components` (top-level)

For EACH directory under `components/` that hosts files (Variant A) or
EACH shard of the model (Variant B), declare a matching top-level
`components:` entry. Names MUST match exactly.

Variant A example:

```yaml
components:
  model-{{MODEL_SLUG}}-gguf:
    type: standard
    summary: {{MODEL_DISPLAY_NAME}} {{QUANT}} GGUF
    description: Model weights for {{MODEL_DISPLAY_NAME}} in GGUF format

  mmproj-{{MMPROJ_SLUG}}-gguf:            # only if multimodal
    type: standard
    summary: {{MODEL_DISPLAY_NAME}} multimodal projector GGUF
    description: Multimodal projector weights for {{MODEL_DISPLAY_NAME}}

  llamacpp:
    type: standard
    summary: llama.cpp using default CPU instruction sets
    description: LLM inference in C/C++

  llamacpp-cuda:
    type: standard
    summary: llama.cpp with CUDA backend
    description: LLM inference in C/C++

  llamacpp-rocm:                          # only if AMD/ROCm engine declared
    type: standard
    summary: llama.cpp with ROCm backend
    description: LLM inference in C/C++
```

Variant B example (use YAML anchor to dedupe shard entries):

```yaml
components:
  model-{{MODEL_SLUG}}-1-of-{{N_SHARDS}}: &model
    type: standard
    summary: Sharded {{MODEL_DISPLAY_NAME}} {{QUANT}}
    description: Model weights for {{MODEL_DISPLAY_NAME}} in GGUF format
  model-{{MODEL_SLUG}}-2-of-{{N_SHARDS}}:
    <<: *model
  # ...up to shard N
```

All `type: standard`. No other component types observed.

---

## 4. Hook conventions

### 4.1 `snap/hooks/install`

MUST be a `#!/bin/bash -eu` script that:

1. Tags syslog output:

```bash
tag="snap.$SNAP_INSTANCE_NAME.hook.install"
exec 1> >(tee >(logger --tag=$tag))
exec 2> >(logger --stderr --priority error --tag=$tag)
```

2. Seeds default configuration via `modelctl`:

```bash
modelctl set --package verbose="false"
modelctl set --package http.port="{{PORT}}"
modelctl set --package http.host="127.0.0.1"
modelctl set --package webui.http.port="{{WEBUI_PORT}}"
modelctl set --package webui.http.host="127.0.0.1"
```

`{{PORT}}` and `{{WEBUI_PORT}}` MUST be distinct, MUST be in the
unprivileged TCP range, and SHOULD use a snap-unique pair (existing
pairs: 8334/8335 nemotron, 8336/8337 gemma4 — pick a different pair for
each new snap to avoid bind clashes on multi-snap hosts).

3. (If `layout` for WSL is declared) creates the WSL symlinks:

```bash
ln -s /var/lib/snapd/hostfs/usr/lib/wsl/lib $SNAP_COMMON/usr/lib/wsl/lib
ln -s /var/lib/snapd/hostfs/usr/lib/wsl/drivers $SNAP_COMMON/usr/lib/wsl/drivers
```

4. Auto-selects an engine:

```bash
if snapctl is-connected hardware-observe; then
    modelctl use-engine --auto --assume-yes --verbose
elif lscpu -V &> /dev/null; then
    echo "Able to access hardware info without hardware-observe interface connection: assuming dev mode installation."
    modelctl use-engine --auto --assume-yes --verbose
else
    echo "hardware-observe interface not auto connected. Skip auto engine selection."
fi
```

Exit code MUST be 0 on success and on the "skip" branch. Failing the
auto-select MUST NOT block install if the interface is simply not
connected.

### 4.2 `snap/hooks/post-refresh`

MUST be a `#!/bin/bash -eu` script that:

1. Tags syslog output (same pattern as `install`, with `hook.post-refresh`).
2. (If WSL layout) re-creates symlinks with `ln -sfn` (force).
3. Runs `modelctl use-engine --fix --assume-yes --verbose` guarded by the
   same `hardware-observe` / `lscpu -V` fallback ladder as `install`.

MUST NOT re-seed `modelctl set --package` defaults — those persist
across refreshes.

---

## 5. Script conventions (`scripts/`)

### 5.1 `scripts/server.sh`

MUST be the daemon entrypoint. Two acceptable forms; both `exec` into
the active engine's `server`:

Form A (uses `modelctl show-engine`, and runs `export-shared-configs.sh`
when content slots are declared):

```bash
#!/bin/bash
set -euo pipefail
$SNAP/bin/export-shared-configs.sh        # only if status/owui slots exist
engine="$(modelctl show-engine --format=json | jq -r .name)"
exec modelctl run -- "$SNAP/engines/$engine/server" "$@"
```

Form B (uses `modelctl status --wait-for-components` and passes
`--wait-for-components` to `modelctl run`):

```bash
#!/bin/bash
set -euo pipefail
engine="$(modelctl status --wait-for-components --format=json | jq -r .engine)"
exec modelctl run --wait-for-components -- "$SNAP/engines/$engine/server" "$@"
```

Both MUST `exec` (no trailing logic). Both depend on `jq` (provided by
`scripts` part `stage-packages`).

### 5.2 `scripts/server-webui.sh`

MUST be:

```bash
#!/bin/bash
set -euo pipefail
port="$(modelctl get webui.http.port)"
host="$(modelctl get webui.http.host)"
capabilities="{{WEBUI_CAPABILITIES}}"     # e.g. "text, text:markdown, vision"
exec modelctl serve-webui "{{WEBUI_ASSETS}}" --port "$port" --host "$host" --capabilities "$capabilities"
```

`{{WEBUI_ASSETS}}` SHOULD be `"$SNAP/webui"` directly; alternatively set
`UI_ASSETS_DIRECTORY: $SNAP/webui` in the `server-webui` app environment
and reference `"$UI_ASSETS_DIRECTORY"` here.

Capabilities MUST be a comma-space-separated list drawn from
`{text, text:markdown, vision, audio}`. Vision MUST appear iff the snap
ships an `mmproj` component.

### 5.3 `scripts/export-shared-configs.sh` (only with content slots)

MUST write a JSON status file under `$STATUS_SHARE` and an OpenAI
endpoint descriptor under `$OWUI_SHARE`:

```bash
#!/bin/bash -eu
status_json=$(modelctl status --format=json --wait-for-components)
mkdir -p "$STATUS_SHARE"
echo "$status_json" > "$STATUS_SHARE/status.json"
rm -f "$OWUI_SHARE/openai.json"
openai_url=$(echo "$status_json" | jq -r '.endpoints.openai // empty')
if [ -n "$openai_url" ]; then
  mkdir -p "$OWUI_SHARE"
  jq -n --arg base_url "$openai_url" '{"base_url": $base_url}' > "$OWUI_SHARE/openai.json"
fi
```

### 5.4 `scripts/completion.bash`

MUST be verbatim:

```bash
# Bash completion for the app named after the snap
unset -f _init_completion
source <($SNAP/bin/modelctl completion bash)
```

---

## 6. `engines/<name>/engine.yaml` schema

### 6.1 Required keys

```yaml
name: {{ENGINE_NAME}}                     # MUST equal the directory name
description: {{ONE_LINE_HUMAN_DESCRIPTION}}
vendor: Canonical Ltd
grade: stable                             # or devel; MUST match snapcraft.yaml grade or be lower
devices:
  # at least one of allof: / anyof: required (see 6.2)
memory: {{N}}G                            # MUST, integer GB, RAM headroom for the engine
disk-space: {{N}}G                        # MUST, integer GB, on-disk weights footprint
components:                               # MUST
  - {{runtime-component}}                 # exactly one llamacpp/llamacpp-cuda/llamacpp-rocm
  - {{model-component}}                   # 1 entry (Variant A) or N entries (Variant B, all shards)
  - {{mmproj-component}}                  # if multimodal
configurations:
  sleep-idle-seconds: 600                 # SHOULD, default idle shutdown
```

Every name in `components:` MUST also appear in `snap/snapcraft.yaml`
top-level `components:`. Static-checks blocks on any mismatch.

### 6.2 `devices:` matchers

CPU-only engine:

```yaml
devices:
  anyof:
    - type: cpu
      architecture: amd64
    - type: cpu
      architecture: arm64
```

NVIDIA GPU engine (note `allof` for the GPU + `anyof` for arch):

```yaml
devices:
  allof:
    - type: gpu
      vendor-id: 0x10de
  anyof:
    - type: cpu
      architecture: amd64
    - type: cpu
      architecture: arm64
```

AMD GPU engine (allof CPU = amd64, then anyof list of supported gfx
microarchitectures):

```yaml
devices:
  allof:
    - type: cpu
      architecture: amd64
  anyof:
    - type: gpu
      vendor-id: 0x1002
      microarchitecture: gfx1010
    # ...repeat for each supported microarchitecture
```

Canonical AMD `microarchitecture` list (use this set unless the user
specifies otherwise): `gfx1010, gfx1011, gfx1012, gfx1030, gfx1031,
gfx1032, gfx1100, gfx1101, gfx1102, gfx1150, gfx1151, gfx1152, gfx1153,
gfx1200, gfx1201`.

### 6.3 Memory/disk sizing rules

- `memory` SHOULD be at least the quantized GGUF size for fully-loaded
  inference, OR the projected resident-set with `mmap` (smaller). When
  mmap is relied upon, add a `# The model will not fit into this much
  RAM, but it is memory mapped` comment.
- `disk-space` MUST be >= sum of weights and runtime component
  (uncompressed), and SHOULD be padded by ~30% for snapd snapshots when
  the model is large.

### 6.4 `engines/<name>/server`

MUST be a minimal `#!/bin/bash -eu` that reads idle config and execs the
matching runtime component server:

```bash
#!/bin/bash -eu
sleep_idle_seconds="$(modelctl get sleep-idle-seconds)"
exec "$SNAP_COMPONENTS/{{RUNTIME_COMPONENT}}/server" --sleep-idle-seconds "$sleep_idle_seconds" "$@"
```

A multimodal engine MAY also pass `--mmproj "$MMPROJ_FILE"` explicitly,
and a CUDA engine MAY apply a `-fitt` VRAM headroom workaround (see
Nemotron CUDA engine for the canonical example: free_vram = 1024 MiB +
mmproj size MiB).

### 6.5 Engine naming

The directory name and `name:` field MUST agree. Conventional names:

- `cpu`, `cpu-<size>` (e.g. `cpu-e4b`, `cpu-26b`)
- `nvidia-gpu`, `nvidia-gpu-<size>`
- `amd-gpu`, `amd-gpu-<size>`

Use the size suffix iff multiple model sizes share one snap.

---

## 7. `components/<name>/component.yaml` schema

### 7.1 Runtime (llama.cpp) component

```yaml
servers:
  openai:
    protocol: http
    base-path: /v1

environment:
  - PATH=$PATH:$COMPONENT/bin
  - LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$COMPONENT/lib
  - LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$COMPONENT/usr/lib/$ARCH_TRIPLET
```

`servers.openai` MUST be present so the inference framework knows this
component exposes an OpenAI-compatible HTTP API at `/v1`.

The runtime component MUST also ship a `server` script. Minimal form:

```bash
#!/bin/bash -eu
port="$(modelctl get http.port)"
host="$(modelctl get http.host)"
mmproj_args=()
if [ -n "${MMPROJ_FILE:-}" ]; then
  mmproj_args=(--mmproj "$MMPROJ_FILE")
fi
set -x
exec llama-server \
  --model "$MODEL_FILE" \
  --alias "$MODEL_NAME" \
  "${mmproj_args[@]}" \
  --port "$port" \
  --host "$host" \
  --no-warmup \
  "$@"
```

The `--alias "$MODEL_NAME"` flag SHOULD be present so that
`/v1/models` returns a stable, human-readable id matching
`{{MODEL_ALIAS}}` (see 8).

A `verbose` toggle MAY be added:

```bash
verbose="$(modelctl get verbose)"
EXTRA_ARGS=()
if [ "${verbose}" = "true" ]; then EXTRA_ARGS+=(--verbose); fi
```

### 7.2 Model component (Variant A, single file)

```yaml
environment:
  - MODEL_FILE=$COMPONENT/{{MODEL_FILE_BASENAME}}.gguf
  - MODEL_NAME={{MODEL_ALIAS}}
```

The basename MUST exactly match the file present in the component dir.
Static-checks blocks on mismatch.

### 7.3 Model component shard 1 (Variant B)

```yaml
environment:
  - SHARDS_DIR=/tmp/{{MODEL_SLUG}}-shards
  - MODEL_FILE=$SHARDS_DIR/{{FILE_PREFIX}}-00001-of-{{N_SHARDS_PADDED}}.gguf
  - MODEL_NAME={{MODEL_ALIAS}}            # MAY be omitted if runtime server does not pass --alias

layout:
  $SHARDS_DIR/{{FILE_PREFIX}}-00001-of-{{N_SHARDS_PADDED}}.gguf:
    symlink: $SNAP_COMPONENTS/{{MODEL_SLUG}}-1-of-{{N_SHARDS}}/{{FILE_PREFIX}}-00001-of-{{N_SHARDS_PADDED}}.gguf
  $SHARDS_DIR/{{FILE_PREFIX}}-00002-of-{{N_SHARDS_PADDED}}.gguf:
    symlink: $SNAP_COMPONENTS/{{MODEL_SLUG}}-2-of-{{N_SHARDS}}/{{FILE_PREFIX}}-00002-of-{{N_SHARDS_PADDED}}.gguf
  # ...one entry per shard, all pointing at $SNAP_COMPONENTS/.../<shardN>.gguf
```

`{{N_SHARDS_PADDED}}` is the zero-padded total (e.g. `00004`, `00006`)
that matches the upstream GGUF filename convention.

### 7.4 Model component shards 2..N (Variant B)

Each MUST have a `component.yaml`. Content MAY be a single comment line:

```yaml
# Used in shards different from the first shard
```

Without this file, snapcraft refuses to assemble the component.

### 7.5 Multimodal projector component

```yaml
environment:
  - MMPROJ_FILE=$COMPONENT/{{MMPROJ_FILE_BASENAME}}.gguf
```

No `MODEL_NAME` here. `MMPROJ_FILE` is consumed by the runtime
component's `server` script (see 7.1).

---

## 8. Naming convention table

| Substitution | Meaning | Example | Constraints |
| --- | --- | --- | --- |
| `{{SNAP_NAME}}` | top-level `name:` | `gemma4`, `nemotron-3-nano-omni` | lowercase, hyphens; matches CLI app + `bin/<name>` symlink |
| `{{MODEL_DISPLAY_NAME}}` | human-readable model name | `Google Gemma 4 E4B` | used only in prose fields |
| `{{MODEL_SLUG}}` | component-naming token | `e4b-q4-k-m-gguf`, `30b-a3b-reasoning-q4-k-m` | lowercase, hyphens; appears in component names and dir names |
| `{{MODEL_ALIAS}}` | `MODEL_NAME` env value | `gemma4-e4b-q4-k-m` | MUST be the string returned by `/v1/models`; MUST be referenced verbatim by build-and-prompt-check |
| `{{MODEL_FILE_BASENAME}}` | exact GGUF filename minus `.gguf` | `gemma-4-E4B-it-Q4_K_M` | MUST match the file on disk byte-for-byte |
| `{{MMPROJ_FILE_BASENAME}}` | mmproj GGUF basename | `mmproj-gemma-4-E4B-it-Q8_0` | as above |
| `{{QUANT}}` | quantization label | `Q4_K_M`, `BF16`, `Q8_0` | for component summaries |
| `{{PORT}}` | OpenAI HTTP port | `8336` | MUST be unique per snap on a host; pair with `{{WEBUI_PORT}}` |
| `{{WEBUI_PORT}}` | WebUI HTTP port | `8337` | MUST != `{{PORT}}` |
| `{{ENGINE_NAME}}` | engine dir + `name:` | `nvidia-gpu-e4b` | MUST agree across dir, yaml, and `engines/<name>/server` path |
| `{{N_SHARDS}}` | total shards (Variant B) | `4`, `6` | matches actual file count |
| `{{N_SHARDS_PADDED}}` | zero-padded total | `00004`, `00006` | matches GGUF filename suffix |
| `{{FILE_PREFIX}}` | GGUF shard filename prefix | `gemma-4-26B-A4B-it-UD-Q4_K_M`, `nemotron-3-nano-omni-30b-a3b-reasoning-fp8_v1.0-Q4_K_M` | MUST match shard filenames |
| `{{CLI_TAG}}` | `inference-snaps-cli` release tag | `v1.0.0-beta.50` | use latest at scaffold time |
| `{{WEBUI_TAG}}` | `inference-snaps-webui` release tag | `v1.0.0-beta.7` | use latest at scaffold time |
| `{{LLAMACPP_TAG}}` | `llama.cpp-builds` release tag | `b8893` | use latest at scaffold time |
| `{{VERSION_PREFIX}}` | snap version prefix | `e4b`, `v1.0` | free-form; combined with `git describe` |
| `{{WEBUI_CAPABILITIES}}` | comma-space list | `text, text:markdown, vision` | drawn from `text`, `text:markdown`, `vision`, `audio`; `vision` iff mmproj exists |

### Cross-name agreement (audited by static-checks)

The following names MUST agree exactly:

- `apps.{{SNAP_NAME}}.command: bin/{{SNAP_NAME}}` ↔ `parts.cli.override-build` symlink target `bin/{{SNAP_NAME}}`.
- Every name listed in any `engines/*/engine.yaml#components:` ↔ a top-level `components:` entry in `snap/snapcraft.yaml`.
- Every name listed in top-level `components:` ↔ a `parts.*` block that organizes files into `(component/<name>)` OR a `local-component-files`-style mapping that does.
- Every `engines/<dir>/engine.yaml#name` ↔ the parent `<dir>` ↔ the `engines/<dir>/server` path used by `scripts/server.sh`.
- `MODEL_NAME` in the model component env ↔ the model id used in any prompt/smoke test ↔ the `--alias` argument the runtime component's `server` passes to `llama-server`.
- `MODEL_FILE` basename in the model component env ↔ the GGUF file actually committed/downloaded into that component dir.
- `MMPROJ_FILE` basename ↔ the mmproj GGUF actually committed/downloaded.

---

## 9. Required snap interfaces

Auto-connect candidates (declared via `plugs` on apps/hooks):

| Interface | Where | Why |
| --- | --- | --- |
| `hardware-observe` | hooks (install, post-refresh), CLI app, server app | engine auto-selection (`modelctl use-engine --auto`) |
| `opengl` | hooks, CLI app, server app | GPU vendor/library access |
| `network` | CLI app | model fetch/sideload from network |
| `network-bind` | server app, server-webui app | bind HTTP listener |
| `home` (read: all) | top-level plug, server app | sideloaded models from `$HOME` |
| `desktop` | CLI app | open WebUI in browser |
| `process-control` | server app | MAY; daemon lifecycle controls |

`process-control` is observed only in the gemma4 single-file snap and is
optional.

---

## 10. Variant B sharding pattern — minimal recipe

When the model exceeds 5 GB and MUST be sharded:

1. Decide `{{N_SHARDS}}` (driven by upstream GGUF split or `gguf-split`
   output). Conventional widths: 4–6.
2. Place shards under `components/{{MODEL_SLUG}}/` named
   `{{FILE_PREFIX}}-NNNNN-of-{{N_SHARDS_PADDED}}.gguf`.
3. In `snap/snapcraft.yaml`:
   - Add N entries to top-level `components:` (use YAML anchor).
   - Add a `model-{{MODEL_SLUG}}` part (Form B1) OR add explicit
     `organize` lines to `local-component-files` (Form B2). For Form B2,
     also commit `component-2-of-N.yaml`...`component-N-of-N.yaml`
     placeholder files alongside the shards.
4. In `components/{{MODEL_SLUG}}/component.yaml` (shard 1) declare the
   `layout` block enumerating all shards as symlinks under
   `$SHARDS_DIR=/tmp/{{MODEL_SLUG}}-shards`.
5. Set `MODEL_FILE` to the shard 1 symlink path (NOT the
   `$SNAP_COMPONENTS/.../<shard1>.gguf` path).
6. Update every relevant `engines/*/engine.yaml` to list ALL N shard
   components in its `components:` list. Missing any shard MUST be
   treated as a blocking error.

---

## 11. Common blocking failures (and the structural rule that prevents each)

| Failure | Preventing rule |
| --- | --- |
| `engine.yaml` references a component not in top-level `components:` | 3.11 + 6.1: every engine component name MUST appear in `snap/snapcraft.yaml#components`. |
| Component prime dir is empty / lacks `component.yaml` | 3.9 + 7.4: every component dir MUST have `component.yaml`; shard 2..N MAY use placeholder content. |
| `MODEL_FILE` points at a missing file | 7.2/7.3 + 8: basename MUST match the GGUF on disk. |
| Wrong model name returned by `/v1/models` | 7.1 (runtime `server` passes `--alias "$MODEL_NAME"`) + 7.2 (`MODEL_NAME` set). |
| HTTP server fails to bind | 4.1 sets non-clashing `{{PORT}}`/`{{WEBUI_PORT}}` + 3.10 `network-bind` plug. |
| WebUI cannot find assets | 5.2 `{{WEBUI_ASSETS}}` MUST resolve (either `$SNAP/webui` literal or `$UI_ASSETS_DIRECTORY` env). |
| Engine auto-select fails silently on first install | 4.1 ladder MUST fall back to `lscpu -V` and SHOULD print a skip message. |
| Sharded model fails to mmap because shards live in separate dirs | 7.3 `layout` symlinks MUST flatten all shards into one `$SHARDS_DIR`. |
| `MODEL_NAME` mismatch between component env and smoke/prompt tests | 8 cross-name agreement. |
| ROCm part builds on arm64 and breaks the build | 3.8 ROCm part guards every `override-*` with the `$CRAFT_ARCH_BUILD_FOR == amd64` check. |
| Snap rejected by store for size | Variant selector (section 1): use Variant B when any single GGUF > 5 GB. |
| `bin/{{SNAP_NAME}}` symlink missing | 3.7 `cli` part `override-build` MUST create `ln --symbolic ./modelctl bin/{{SNAP_NAME}}`. |
| `completion.bash` not picked up | 3.10 CLI app MUST declare `completer: bin/completion.bash`. |
| Hooks lack hardware access | 3.6 install + post-refresh MUST declare `hardware-observe` and `opengl`. |

---

## 12. Scaffolding algorithm (deterministic)

Given user inputs:
- `MODEL_DISPLAY_NAME`, `MODEL_SLUG`, `MODEL_ALIAS`, `MODEL_FILE_BASENAME`
- target `PORT`, `WEBUI_PORT`
- whether multimodal (yes → `mmproj` component, vision capability)
- on-disk size of primary GGUF

1. Select Variant A iff every GGUF is < 5 GB; else Variant B.
2. Create the directory skeleton from section 2.
3. Generate `snap/snapcraft.yaml` from 3.1–3.11, picking runtime backends
   (CPU MUST be present; NVIDIA SHOULD be present; AMD MAY be present).
4. Generate `snap/hooks/install` and `post-refresh` from section 4.
5. Generate `scripts/server.sh`, `server-webui.sh`, `completion.bash`,
   and (if content slots) `export-shared-configs.sh` from section 5.
6. For each backend × model-size combination, generate an
   `engines/<name>/engine.yaml` + `server` from section 6.
7. For each component (runtime + model + optional mmproj), generate
   `components/<name>/component.yaml` + (runtime only) `server` from
   section 7.
8. For Variant B, additionally generate shard-2..N placeholder
   `component.yaml` files.
9. Run static-checks against the cross-name agreement table in section
   8 and the blocking-failure table in section 11.
