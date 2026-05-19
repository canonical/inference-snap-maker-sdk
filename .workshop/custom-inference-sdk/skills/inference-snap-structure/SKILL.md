---
name: inference-snap-structure
description: Create and adapt the inference snap structure from reference examples (packaging, engines, hooks, scripts, components).
trigger: Keywords like "create snap structure", "adapt inference snap", "scaffold from gemma4-snap", "set up engines/hooks/components"
scope: user
---

# Inference Snap Structure

## Purpose

Create the snap structure by reusing proven patterns from example repositories, with minimal targeted adaptation.

## Primary References

- Main example: https://github.com/canonical/gemma4-snap
- Sharding reference only when model is larger than 5 GB: https://github.com/canonical/nemotron-3-nano-omni-snap

## Workflow

1. Ask for the target workspace path.
2. Ask whether the target repo already exists or must be created.
3. Ask for ports/hosts before editing hooks.
4. Map structure from the example:
   - snap/snapcraft.yaml
   - snap/hooks/install
   - snap/hooks/post-refresh
   - scripts/server.sh
   - scripts/server-webui.sh
   - engines/*/engine.yaml and engines/*/server
   - components/*/component.yaml
5. Adapt names, model IDs, component names, engine names, and startup scripts.
6. Keep model/engine/hook/script responsibilities separated.
7. If model > 5 GB, apply sharding logic using the Nemotron sharding pattern only for sharding-related parts.
8. Do not remove existing variants unless explicitly requested.

## Output

- File map of copied vs adapted files.
- Rationale for naming/layout decisions.
- Sharding decision (required or not) with reason.
- Any assumptions and missing inputs.

## Rules

- Prefer minimal changes over redesign.
- Do not introduce new dependencies/download flows without confirmation.
- Do not skip user-provided port and host requirements.
