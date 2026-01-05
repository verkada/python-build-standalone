# python-build-standalone (Verkada Fork)

This is Verkada's fork of `astral-sh/python-build-standalone`.

## Syncing with Upstream

This repo has two remotes:
- `origin`: `verkada/python-build-standalone` (our fork)
- `upstream`: `astral-sh/python-build-standalone` (source repo)

To sync with upstream:

```bash
# Fetch latest from upstream
git fetch upstream

# Create a branch for the sync (main is protected)
git checkout main
git checkout -b nickvines/sync-upstream-main

# Merge upstream changes
git merge upstream/main --no-edit

# Push and create PR
git push -u origin nickvines/sync-upstream-main
```

## Creating PRs

**IMPORTANT**: Always create PRs against `verkada/python-build-standalone`, NOT the upstream repo.

```bash
# Correct - targets our fork
gh pr create --repo verkada/python-build-standalone --title "Title" --body "Body"

# WRONG - would target upstream
gh pr create --title "Title" --body "Body"
```

The default `gh pr create` may target the upstream repo since this is a fork. Always explicitly specify `--repo verkada/python-build-standalone`.
