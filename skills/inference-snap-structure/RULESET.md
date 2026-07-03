# Inference Snap Structure RULESET (v2, gemma4-aligned)

This ruleset is self-sufficient. Given a user request like
"snap up model X with HTTP port P and WebUI port Q" plus the substitution
table at the end, an agent MUST be able to scaffold a Canonical inference snap
without external examples.

Terminology: **MUST** = required for build/runtime correctness; **SHOULD** =
recommended convention; **MAY** = optional.

This ruleset is aligned to the v2 layout used by `canonical/gemma4-snap`:
- model metadata is in `models/*/model.yaml`
- runtime metadata is in `runtimes/*/runtime.yaml`
- engines select `runtime` + `model`, not explicit component lists

---

## 1. Variant selector (read first)

Choose ONE packaging variant for each model artifact set:

- **Variant A - single component model.**
  One model component contains all model files.
  Use when the component payload is comfortably below Store limits.

- **Variant B - split/sharded model.**
  Model is split across multiple components because a single component would be
  too large (soft threshold around 5 GB in practice).
  Typical examples:
  - GGUF shards `...-00001-of-00004.gguf` etc.
  - OpenVINO/IR split across `...-1-of-2`, `...-2-of-2` components.

### Shard sizing (MUST)

- Each component MUST be **< 5 GB** (hard Store limit per component).
- First measure the real artifact size (bytes). For a HuggingFace GGUF, use the
  `x-linked-size` header of a redirect HEAD request
  (`curl -sIL "$url" | grep -i x-linked-size`); fall back to the final
  `content-length`.
- Choose the shard count as `n_shards = ceil(size / 4.8GB)`. The 4.8 GB target
  leaves margin below 5 GB. Example: an 18.3 GB model needs `ceil(18.3/4.8) = 4`
  shards (~4.6 GB each); **3 shards would be ~6.1 GB each and is INVALID**.
- Shards MUST be **valid, independently-parseable GGUF files** produced with
  `llama-gguf-split --split --split-max-size <N>M`, which yields the
  `...-00001-of-000NN.gguf` naming. Do **NOT** use a raw byte split
  (`split -b`): raw fragments are not loadable by llama-server and would require
  a reassembly step the runtime does not perform.
- llama-server is pointed at shard 1 only (`MODEL_FILE=...-00001-of-000NN.gguf`);
  it auto-discovers the remaining shards from the same directory, so `model.yaml`
  MUST symlink every shard into one common `SHARDS_DIR` (see §8.2).

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
    export-shared-configs.sh             # MAY (only if status/owui slots declared)
  Makefile                               # SHOULD (used to download models)
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

## 3. snap/snapcraft.yaml shape

### 3.1 Top-level fields

```yaml
name: {{SNAP_NAME}}
base: core24
summary: Inference Snap for {{MODEL_DISPLAY_NAME}}
description: |
  ...
adopt-info: version
grade: stable
confinement: strict
compression: lzo
assumes:
  - snapd2.68
platforms:
  amd64:
  arm64:
```

SHOULD include `website`, `source-code`, and `issues` fields. Also
SHOULD include `title`, `contact`, and `license` — snapcraft's metadata linter
warns when they are empty.

### 3.2 Environment

MUST include:

```yaml
environment:
  SNAP_COMPONENTS: /snap/$SNAP_INSTANCE_NAME/components/$SNAP_REVISION
  ARCH_TRIPLET: $CRAFT_ARCH_TRIPLET_BUILD_FOR
```

MAY include content-sharing paths (if matching slots exist):

```yaml
  STATUS_SHARE: *status-share
  OWUI_SHARE: *owui-share
```

MAY include OpenCL env when Intel/OpenCL runtime is used:

```yaml
  OCL_ICD_VENDORS: $SNAP/etc/OpenCL/vendors
```

### 3.3 Plugs and slots

MUST declare home plug for sideloading:

```yaml
plugs:
  home:
    read: all
```

MAY declare content slots (status + Open WebUI integration):

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

### 3.4 Layout (optional)

WSL workaround MAY be declared:

```yaml
layout:
  /usr/lib/wsl:
    bind: $SNAP_COMMON/usr/lib/wsl
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
- `version` (computed from CLI version + git hash, or equivalent deterministic versioning)
- `cli` (inference-snaps-cli tarball for amd64 and arm64)
- `cli-dependencies` (typically includes `pciutils`; add `clinfo` for Intel GPU detection)
- `webui`
- `engines`
- `models`
- `runtimes`
- `scripts` (with `jq` stage package)
- `local-component-files`
- runtime payload parts (for example `llamacpp`, `llamacpp-cuda`, `llamacpp-rocm`, `openvino-model-server`)

`local-component-files` SHOULD use `plugin: cmake` with an `override-build` copy
workaround to avoid unwanted `dump` behavior for very large payloads, and MUST
end with `prime: [-*]` so unorganized component sources do not leak into the base
snap. ADDED — canonical shape (from gemma4-snap):

```yaml
  local-component-files:
    plugin: cmake
    source: components
    override-build: |
      cp -rf --archive --link --no-dereference ${CRAFT_PART_SRC}/* ${CRAFT_PART_INSTALL}
    organize:
      "model-<slug>/*": (component/model-<slug>)
      "mmproj-<slug>/*": (component/mmproj-<slug>)
      # one line per shard for split models, mapping each shard file to its component
    prime:
      - -*   # exclude everything not explicitly organized
```

NVIDIA detection SHOULD use a minimal `cli-nvidia-smi` part that copies only
`/usr/bin/nvidia-smi` and related license text.

**Runtime & CLI artifact sources** (pin a known-good tag; verify each URL
returns HTTP 200 before use). A literal scaffold with placeholder parts will NOT
build — use these real sources:

- CLI: `https://github.com/canonical/inference-snaps-cli/releases/download/<CLI_TAG>/inference-snaps-cli-linux-{amd64,arm64}.tar.xz`
  (ships `bin/modelctl` and `bin/snap-completer.bash`). Symlink the app command
  to `modelctl` in `override-build`: `ln --symbolic ./modelctl bin/<SNAP_NAME>`.
- WebUI: `https://github.com/canonical/inference-snaps-webui/releases/download/<WEBUI_TAG>/inference-snaps-webui.tar.xz`, organized into `webui/`.
- llama.cpp runtimes: `https://github.com/canonical/llama.cpp-builds/releases/download/<LLAMA_BUILD>/llamacpp-{amd64,arm64}[+cuda12.9|+rocm|+onemkl].tar.gz`,
  each organized into `(component/<runtime>)` with `stage-packages: [libgomp1]`
  (add `libatomic1` for rocm/onemkl).

The same `<CLI_TAG>` MUST be used by the `cli` part AND by the
`validate-engines` CI job's checkout `ref`.

### 3.7 Components (top-level)

Every component payload under `components/` MUST have a matching top-level
`components:` entry in `snapcraft.yaml`.

All component entries MUST be `type: standard`.

For split/sharded models, use YAML anchors to avoid duplication:

```yaml
components:
  model-{{MODEL_SLUG}}-1-of-{{N_SHARDS}}: &model
    type: standard
    summary: ...
    description: ...
  model-{{MODEL_SLUG}}-2-of-{{N_SHARDS}}:
    <<: *model
```

### 3.8 Apps

MUST define three apps:

```yaml
apps:
  {{SNAP_NAME}}:
    command: bin/{{SNAP_NAME}}
    completer: bin/{{COMPLETER_FILE}}
    plugs:
      - hardware-observe
      - opengl
      - network
      - desktop
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
      # MAY include process-control

  server-webui:
    command: bin/server-webui.sh
    daemon: simple
    plugs:
      - network-bind
```

`{{COMPLETER_FILE}}` MAY be `completion.bash` (repo script) or another file
shipped by the CLI release (for example `snap-completer.bash`).

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
4. If WSL layout is used, create symlinks for `/usr/lib/wsl/lib` and
   `/usr/lib/wsl/drivers` via `$SNAP_COMMON`.
5. Auto-select engine with non-interactive fallback:

```bash
modelctl use-engine --auto --assume-yes --fallback=cpu
```

`--components` MAY be added for repos that require component-aware selection.

### 4.2 snap/hooks/post-refresh

MUST:
1. Use same syslog tagging pattern.
2. Recreate WSL symlinks with `ln -sfn` when layout is used.
3. Refresh active engine:

```bash
modelctl use-engine --fix --assume-yes --fallback=cpu
```

MUST NOT re-seed package defaults on refresh.

---

## 5. Script conventions

### 5.1 scripts/server.sh

MUST select active engine and execute its server script:

```bash
#!/bin/bash
set -euo pipefail
engine="$(modelctl show-engine --format=json | jq -r .name)"
exec modelctl run -- "$SNAP/engines/$engine/server" "$@"
```

If content-sharing slots are used, SHOULD run
`$SNAP/bin/export-shared-configs.sh` before launching engine.

### 5.2 scripts/server-webui.sh

MUST obtain host/port from modelctl and run webui service:

```bash
#!/bin/bash
set -euo pipefail
port="$(modelctl get webui.http.port)"
host="$(modelctl get webui.http.host)"
capabilities="{{WEBUI_CAPABILITIES}}"
exec modelctl serve-webui "$SNAP/webui" --port "$port" --host "$host" --capabilities "$capabilities"
```

### 5.3 scripts/export-shared-configs.sh (if content slots)

MUST write status JSON and OpenAI endpoint descriptor:

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

### 5.4 scripts/completion.bash (OPTIONAL — usually NOT needed)

The v2 `inference-snaps-cli` release tarball already ships
`./bin/snap-completer.bash`. The recommended default is therefore to set the
app `completer: bin/snap-completer.bash` (as `gemma4-snap` does) and NOT ship a
repo `scripts/completion.bash` at all.

Only add a repo `scripts/completion.bash` if you deliberately want a custom
completer. If you do, it is a static source file with this content (it sources
the CLI's completion output at runtime — do NOT generate it at build time):

```bash
unset -f _init_completion
source <($SNAP/bin/modelctl completion bash)
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
`model.default`. Encode the size in the *model id/name* (e.g.
`qwen-3-5-4b-q4-k-xl`, `qwen-3-5-9b-q4-k-m`), NOT in the engine name. This is what
`gemma4-snap` does (3 sizes under a single `cpu` engine) and it keeps
`use-engine --fallback=cpu` valid. Do NOT create `cpu-4b`/`cpu-9b` engines.

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
mmproj_args=()
if [ -n "${MMPROJ_FILE:-}" ]; then
  mmproj_args=(--mmproj "$MMPROJ_FILE")
fi
exec llama-server --model "$MODEL_FILE" --alias "$MODEL_NAME" "${mmproj_args[@]}" --port "$port" --host "$host" --no-warmup --sleep-idle-seconds "$sleep_idle_seconds" "$@"
```

For OpenVINO runtimes, server script typically runs `ovms` using `MODEL_PATH`
and `MODEL_NAME`.

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

---

## 8. Model schema (models/<id>/model.yaml)

### 8.1 Required keys

```yaml
id: {{MODEL_ID}}
name: {{MODEL_FAMILY_OR_SIZE}}
description: {{HUMAN_DESCRIPTION}}
model-card-url: {{URL}}
quantization: {{QUANT_LABEL}}
disk-size: {{SIZE}}   # CHANGED: integer + binary unit only, e.g. 3420M / 6300M / 16163M.
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

For sharded/split models, set an intermediate directory and flatten with layout:

```yaml
environment:
  - SHARDS_DIR=/tmp/{{MODEL_SLUG}}-shards
  - MODEL_FILE=$SHARDS_DIR/{{SHARD_1_FILE}}
  - MODEL_NAME={{MODEL_ALIAS}}
layout:
  $SHARDS_DIR/{{SHARD_1_FILE}}:
    symlink: $SNAP_COMPONENTS/{{COMPONENT_1}}/{{SHARD_1_FILE}}
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
| A shard is ≥ 5 GB (Store rejects the component) | Section 1 shard sizing: `n_shards = ceil(size / 4.8GB)` |
| Shards produced with raw `split -b` are not loadable GGUFs | Section 1 shard sizing: use `llama-gguf-split` |
| Wrong model id in `/v1/models` | `MODEL_NAME` plus `--alias` in llama server |
| Auto-selection breaks install path | Section 4 with `--fallback=cpu` |
| WebUI not reachable | ports seeded in install + `network-bind` on `server-webui` |
| Content-sharing files missing | Section 5.3 export script |

---

## 11. Deterministic scaffolding algorithm

Given:
- model ids and artifacts
- target ports
- capabilities
- desired backends (cpu/nvidia/amd/intel/openvino)
- component size constraints

1. Choose Variant A or B per model payload size.
2. Create v2 skeleton (`engines/`, `models/`, `runtimes/`, `components/`, `scripts/`, `snap/`).
3. Generate `snap/snapcraft.yaml` with required parts/apps/components.
4. Generate hooks with config seeding and engine select/fix commands.
5. Generate `scripts/server.sh` and `scripts/server-webui.sh`.
6. Generate each runtime descriptor (`runtime.yaml`).
7. Generate each model descriptor (`model.yaml`), including layout for split models.
8. Generate each engine descriptor and matching engine `server` script.
9. Verify naming agreements from section 9.
10. Run static checks and fail on any mismatch.

---

## 12. Substitution table

| Substitution | Meaning | Example |
| --- | --- | --- |
| `{{SNAP_NAME}}` | snap/app name | `gemma4`, `fastcontext-1-0` |
| `{{MODEL_ID}}` | `models/<id>/` identifier | `e4b-q4-k-m-gguf` |
| `{{MODEL_ALIAS}}` | runtime model id (API visible) | `gemma4-e4b-q4-k-m` |
| `{{MODEL_FILE}}` | model file basename | `gemma-4-E4B-it-Q4_K_M.gguf` |
| `{{MMPROJ_FILE}}` | mmproj basename | `mmproj-gemma-4-E4B-it-Q8_0.gguf` |
| `{{RUNTIME_NAME}}` | runtime descriptor name | `llamacpp`, `openvino-model-server` |
| `{{RUNTIME_COMPONENT}}` | runtime component payload | `llamacpp`, `openvino-model-server` |
| `{{ENGINE_NAME}}` | engine directory and `name` | `cpu`, `nvidia-gpu`, `intel-gpu` |
| `{{PORT}}` | inference HTTP port | `8336` |
| `{{WEBUI_PORT}}` | webui HTTP port | `8337` |
| `{{WEBUI_CAPABILITIES}}` | webui capability list | `text, text:markdown, vision` |
| `{{N_SHARDS}}` | number of split artifacts | `4` |
| `{{COMPLETER_FILE}}` | app completer path basename | `completion.bash` |

