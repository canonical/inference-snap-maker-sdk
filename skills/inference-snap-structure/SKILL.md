---
name: inference-snap-structure
description: Create and adapt the inference snap structure from reference examples (packaging, engines, hooks, scripts, components).
trigger: Keywords like "create snap structure", "adapt inference snap", "scaffold inference snap", "set up engines/hooks/components"
scope: user
---

# Inference Snap Structure

## Purpose

Create the snap structure by reusing proven patterns from example repositories, with minimal targeted adaptation.

## Primary Reference

- `RULESET.md` (sibling file in this skill directory) — the authoritative structural ruleset extracted from the original example repos. Read it before scaffolding. It defines Variant A (single-file model) and Variant B (sharded model > 5 GB) and is self-contained — do NOT consult external example repos.

## Workflow

1. Ask for the target workspace path.
2. Ask whether the target repo already exists or must be created.
3. Ask for ports/hosts before editing hooks.
4. Read `RULESET.md` and pick the variant (A: model ≤ 5 GB, B: model > 5 GB).
5. Scaffold files per RULESET §2 (directory tree) and §12 (scaffolding algorithm):
6. Map structure from the example:
   - .gitmodules (add `dev` submodule targeting https://github.com/canonical/inference-snaps-dev.git)
   - snap/snapcraft.yaml
   - snap/hooks/install, snap/hooks/post-refresh
   - scripts/server.sh, scripts/server-webui.sh (+ helpers per §5)
   - engines/*/engine.yaml and engines/*/server
   - components/*/component.yaml
   - README.md (ensure it includes `git clone --recurse-submodules` instructions)
7. Substitute the placeholders defined in RULESET §8 (`{{SNAP_NAME}}`, `{{MODEL_ALIAS}}`, `{{PORT}}`, …).
8. Keep model/engine/hook/script responsibilities separated.
9. Apply Variant B sharding pattern (RULESET §10) only when model > 5 GB.
10. Do not remove existing variants unless explicitly requested.
11. Adapt names, model IDs, component names, engine names, and startup scripts.
12. Keep model/engine/hook/script responsibilities separated.
13. If model > 5 GB, apply sharding logic using the Nemotron sharding pattern only for sharding-related parts.
14. Do not remove existing variants unless explicitly requested.

## Output

- File map of copied vs adapted files.
- Rationale for naming/layout decisions.
- Sharding decision (required or not) with reason.
- Any assumptions and missing inputs.

## Rules

- Prefer minimal changes over redesign.
- Do not introduce new dependencies/download flows without confirmation.
- Do not skip user-provided port and host requirements.
- Always initialize the `dev` git submodule in the new project.
- Always include submodule recursion in README.md clone instructions.
