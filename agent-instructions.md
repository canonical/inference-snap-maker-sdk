# Agent Instructions for Inference Snap Creation
## Purpose
These instructions guide the agent when it needs to create and validate a production-ready inference snap using only local workspace skills, agents, and rulesets.
## Principles
Do not consult external repositories.
Use the unified five-stage workflow in order, back-to-back without stopping between steps:

inference-snap-structure
github-workflows
inference-snap-static-checks
inference-snap-build-and-prompt-check
inference-snap-create-pr

Do NOT pause between skills to wait for user confirmation. Run each skill immediately after the previous one completes, passing findings forward. Only stop if a blocking issue requires user input (e.g. missing required inputs, unresolvable build error).
Goal:
Create a production-ready inference snap.

Inputs:

Download model URL: <MODEL_DOWNLOAD_URL>
Target workspace path: <PATH> (must be a subdirectory, never the root of this project)
API port: <API_PORT>
WebUI port: <WEBUI_PORT>
Bind host: <BIND_HOST>
Snap name: <SNAP_NAME>
GitHub repository URL: <GITHUB_REPO_URL>
Hard requirements:

Use installed local rulesets/skills/agents from `/home/workshop/.agents` as the only source of truth.
Preserve structure and make minimal targeted changes.
Ask for confirmation before introducing new dependencies or architecture changes.
If model size > 5 GB, apply sharding logic according to the local RULESET only for sharding-related parts.
Do not leave empty or placeholder components.
Verify model provenance/signature from source metadata (filename, linked size, etag/hash/checksum if available).
Ensure snapcraft consistency: apps/commands/hooks/components/engines all resolve correctly.
Enforce release metadata in snapcraft.yaml: title, summary, description, license, contact, website, source-code, issues.
Keep hook/server/webui ports aligned with requested values.
Always ask the user for both URLs (`<MODEL_DOWNLOAD_URL>` and `<GITHUB_REPO_URL>`) before running the pipeline; do not infer or reuse automatically.
Execution requirements:

Run full static checks and fix blocking issues before build.
Run snapcraft pack --destructive-mode.
Install with: sudo snap install *.snap *.comp --dangerous
Connect required interfaces (hardware-observe, opengl, network-bind, process-control).
Run hardware + engine checks, then auto-select engine; if needed set the expected engine explicitly.
Validate runtime with:
GET /v1/models
POST /v1/chat/completions (short prompt)
If any step fails, iterate: fix, rebuild, reinstall, retest until passing.
After a successful build, initialize a git repository in the snap subdirectory, commit all snap files, and create a pull request on the provided GitHub repository URL.
After the snap PR is created, open a second PR against https://github.com/canonical/inference-snaps to add the new snap to the list in docs/reference/snaps.md. Read the current content of that file first to match its existing format and ordering, then add an entry for the new snap.
Output format:

Section 1: Files changed (copied vs adapted)
Section 2: Static-check findings by severity (with fixes applied)
Section 3: Build/install/interface results
Section 4: Engine and runtime status
Section 5: Prompt test result (request + response snippet)
Section 6: Remaining risks and release-readiness verdict (PASS/FAIL)
If blocked:

Report exact blocking command and error, propose the smallest viable next action.
When invoked make sure to ask for all required inputs before proceeding.
If not provided ask for:

Download model URL
target workspace path (must be a subdirectory of this project, never the root)
API port
WebUI port
Bind host
Snap name
GitHub repository URL (used to initialize git and open the PR)
Follow all hard requirements, execution requirements, and output format above.

