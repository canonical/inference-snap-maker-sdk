---
name: inference-snap-create-pr
description: Push the current branch to a remote repo and open a pull request with the "trigger-tests" label applied.
trigger: Keywords like "create a pull request for the snap", "open a PR for the snap", "push and create PR", "submit snap PR", "create snap pull request"
scope: user
---

# Inference Snap — Create PR

## Purpose

Push the current branch to a remote repository and open a pull request with the `trigger-tests` label applied.

## Pre-flight

Before doing anything, collect from the user:

- **Remote repo URL** — the GitHub repository URL to push to and create the PR against (e.g. `https://github.com/org/repo`). Always ask and require explicit confirmation in the current run; do not infer from existing remotes.
- **PR base branch** — defaults to `main` if not provided.
- **PR title** — defaults to the most recent commit subject if not provided.
- **PR body** — optional; may be left blank.

## Workflow

1. **Detect current branch** — run `git branch --show-current` from the workspace to confirm the working branch.
2. **Configure remote** — check existing remotes with `git remote -v`. If no remote named `origin` points to the provided URL, add or update it with `git remote set-url origin <url>` (or `git remote add origin <url>`).
3. **Push** — run `git push -u origin <branch>`. Never use `--force`.
4. **Ensure label exists** — run `gh label list --repo <url>` and check for `trigger-tests`. If absent, create it: `gh label create trigger-tests --color 0075ca --repo <url>`.
5. **Create PR** — run `gh pr create --repo <url> --title "<title>" --body "<body>" --base <base> --label trigger-tests`.
6. **Verify label** — run `gh pr view <number> --repo <url> --json labels` and confirm `trigger-tests` is listed.

## Rules

- Never force-push.
- If `gh` is not authenticated, surface the exact error and stop — do not attempt workarounds.
- The label `trigger-tests` must be confirmed present before declaring success.
- Do not merge the PR.

## Output

Report the PR URL, number, title, base branch, and label confirmation.
