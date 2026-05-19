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
5. Consistency checks:
   - Model alias in component environment matches prompt usage.
   - API host/port defaults align with requested values.

## Output

- Findings ordered by severity (blocking/non-blocking).
- Exact file-level fixes required.
- Explicit pass/fail for: snapcraft schema consistency, component completeness, model signature checks.

## Rules

- Treat missing model artifact or mismatched model identity as blocking.
- Treat empty components as blocking unless explicitly intended.
- Do not mark validation passed if any blocking issue remains.
