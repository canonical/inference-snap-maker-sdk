---
name: inference-snap-from-example
description: Compatibility wrapper skill for inference snap work. Use specialized skills for structure creation, static checks, and build/prompt verification.
trigger: Keywords like "inference snap", "packaging snap from example", "adapt example repo", "shard model", "large model snap", "create a snap from example"
scope: user
---

# Inference Snap From Example

## Purpose

This skill is a dispatcher for backward compatibility.

Use these specialized skills instead:

1. `inference-snap-structure`
2. `inference-snap-static-checks`
3. `inference-snap-build-and-prompt-check`

## Usage Routing

1. If the user asks to scaffold/adapt files from an example, use `inference-snap-structure`.
2. If the user asks to validate snapcraft/component/model integrity, use `inference-snap-static-checks`.
3. If the user asks to build/install/test with prompts, use `inference-snap-build-and-prompt-check`.
4. If the user is vague, ask clarifying questions to determine which specialized skill to route to.
5. Ask the user if he wants to proceed with all steps in sequence if the request is broad.

## Primary References

- Main example: https://github.com/canonical/gemma4-snap
- Sharding reference only, when model is larger than 5 GB: https://github.com/canonical/nemotron-3-nano-omni-snap

## Rule

For new work, prefer specialized skills and keep this file as a compatibility entry point.
