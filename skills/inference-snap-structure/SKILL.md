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

- `RULESET.md` (sibling file in this skill directory) — the authoritative structural ruleset extracted from the original example repos. Read it before scaffolding. It defines Variant A (single-file model) and Variant B (split model > 5 GB) and is self-contained — do NOT consult external example repos.

## Inputs

1. The target workspace path (where the inference snap repo will be created or adapted)
2. Whether the target repo already exists (if not, stop and ask the user to create it first)
3. Ports/hosts to use in hooks and server scripts (from pre-flight)

## Workflow

1. Read `RULESET.md` and pick the variant (A: model ≤ 5 GB, B: model > 5 GB).
2. Scaffold files following the `RULESET.md`
3. Substitute the placeholders defined in RULESET §8 (`{{SNAP_NAME}}`, `{{MODEL_ALIAS}}`, `{{PORT}}`, …).

## Output

- File map of copied vs adapted files.
- Rationale for naming/layout decisions.
- Model splitting decision (required or not) with reason.
- Any assumptions and missing inputs.

## Rules

- Prefer minimal changes over redesign.
- Do not introduce new dependencies/download flows without confirmation.
- Do not skip user-provided port and host requirements.
- Always initialize the `dev` git submodule in the new project.
