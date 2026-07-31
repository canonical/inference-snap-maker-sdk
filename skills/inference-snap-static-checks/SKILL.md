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

0. Pre-flight input validation (do first; these are cheap and prevent
   late build failures):
   - `snap-name` MUST match `^[a-z0-9]+(-[a-z0-9]+)*$` (snapd rule). A dot,
     underscore, or uppercase is **blocking** — e.g. `qwen3.5` must become
     `qwen3-5`. `snapcraft pack` rejects invalid names outright.
   - Each `models/*/model.yaml` `disk-size` MUST match `^[0-9]+[KMG]$`
     (integer + binary unit, e.g. `3420M`). Decimals or a trailing `B`
     (`3.4GB`) are **blocking**: they break `modelctl list-models` with
     `strconv.ParseUint: parsing "3.4GB": invalid syntax`.

1. snap/snapcraft.yaml checks:
   - Name/summary/description match target model; `name` passes the regex above.
   - Every component under `components/` is declared in `snapcraft.yaml#components`
     and every declared model/mmproj component has a `components/<name>/` dir.
   - Hooks and scripts referenced by snapcraft actually exist on disk.
   - Engine/component naming consistency.
   - Each component name is all lowercase, hyphens only (no underscores/dots).
   - The `organize` step moves files into a correct `(component/<name>)` (or path):
     the component name matches a declared component; for split models there is one
     organize line per model file. `local-component-files` MUST end with
     `prime: [-*]` so unorganized sources don't leak into the base snap.
   - Split models: each part is declared as its own component and organized into
     its own component dir.
   - Each app has the right interfaces (servers need `network-bind`; the `server`
     daemon typically also has `hardware-observe`, `opengl`, `home`, and
     `process-control`).
   - `ADDITIONAL_FEATURES` (non-blocking): on the **main** app only and omits it
     on `server`/`server-webui`. Some CLI versions instead expect `server` to add
     `chat` and `server-webui` to add `webui`. Verify against the CLI version in
     use; treat a mismatch as non-blocking unless feature detection actually fails.

2. Component completeness checks:
   - Each `components/<name>/` dir exists and (after model prep) holds the expected
     GGUF file(s)/part. There is NO `component.yaml`.
   - No placeholder-only component dirs unless intentionally declared (a README-only
     dir is fine pre-download; the artifact must exist before packing).

3. Engine sanity checks:
   - `engine.yaml` exists for each `engines/<name>/` dir and its `server` is
     executable.
   - `engine.yaml` `runtime` resolves to a `runtimes/<runtime>/runtime.yaml`.
   - `engine.yaml` `model.default` and every `model.options` entry resolve to a
     `models/<id>/model.yaml`.
   - Memory/disk constraints (if present) are realistic.
   - Each `engine.yaml` specifies a list of compatible `devices`.

4. Model signature/provenance checks:
   - Verify source model URL and repository owner are the expected target.
   - Capture model filename, linked size (`x-linked-size`), and ETag/hash headers.
   - If available, compare checksums of the downloaded file to trusted metadata.

5. General consistency checks:
   - No duplicate component names.
   - The install hook seeds ports to the requested values and host `127.0.0.1`,
     and its `use-engine --fallback=<engine>` names a real `engines/<engine>/`
     (ADDED: a dangling fallback such as `--fallback=cpu` with only `cpu-4b`/
     `cpu-9b` engines is **blocking**).
   - The `Makefile`/`download-models.sh` downloads the right files to the exact
     `components/<name>/` dirs with the exact filenames referenced by
     `model.yaml` `MODEL_FILE`/`MMPROJ_FILE` and by the snapcraft `organize` map.
     For split models each part lands in its own component dir with the
     `...-000NN-of-000MM.gguf` name llama-server expects.
   - `*.gguf` are git-ignored and NOT pushed via git-lfs.
   - A repo-root `download-models.sh` exists (CI invokes `./download-models.sh`).

6. model.yaml consistency checks:
   - Text/model entries set `MODEL_NAME` (the `--alias`, API-visible id) and
     `MODEL_FILE`. Multimodal entries also set `MMPROJ_FILE`. `capabilities`
     includes `vision` when an mmproj is shipped.
   - Split models set `MODEL_PARTS_DIR` + `MODEL_FILE=$MODEL_PARTS_DIR/<part-1>` and a
     `layout:` block symlinking every part from its component dir into
     `MODEL_PARTS_DIR` (llama-server auto-discovers the rest). mmproj stays a separate
     single-file component.
   - `MODEL_FILE`/`MMPROJ_FILE`/`MODEL_PARTS_DIR` paths resolve at runtime.

7. runtime.yaml consistency checks:
   - Each `runtime.yaml` declares its server protocol(s) (e.g. `openai` `http`
     `/v1`), the `PATH`/`LD_LIBRARY_PATH` env into `$SNAP_COMPONENTS/<runtime>/…`,
     and a `components:` list referencing the runtime payload component(s) that are
     declared in `snapcraft.yaml` and built by parts.
   - The engine `server` file is executable and, for llama.cpp, runs `llama-server
     --model "$MODEL_FILE" --alias "$MODEL_NAME" [--mmproj "$MMPROJ_FILE"] --port …
     --host …`.
   - The model description references the supported silicon and model variant.

8. inference-snaps-cli version checks:
   - The CLI version used in `snap/snapcraft.yaml` matches the pinned `ref:` in
     `validate-engines.yaml` (or the CLI version used to generate the workflows).


## Output

- Findings ordered by severity (blocking/non-blocking).
- Exact file-level fixes required.
- Explicit pass/fail for: snapcraft schema consistency, component completeness, model signature checks.

## Rules

- Treat missing model artifact or mismatched model identity as blocking.
- Treat empty components as blocking unless explicitly intended.
- Do not mark validation passed if any blocking issue remains.
