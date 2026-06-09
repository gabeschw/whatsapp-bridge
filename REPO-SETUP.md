# Repo Setup Summary

## Original Setup

Extracted `whatsapp-bridge/` from the `gabeschw/whatsapp-mcp` monorepo into its own repo using `git filter-repo`, preserving full commit history of just the bridge directory. No root-level project files (README, CI, etc.) were kept — add them as needed.

## Remotes

| Remote | URL | Purpose |
|--------|-----|---------|
| `origin` | `git@github.com:gabeschw/whatsapp-bridge.git` | Your fork (push here) |
| `upstream` | `git@github.com:verygoodplugins/whatsapp-mcp.git` | Source monorepo (pull bridge updates) |

## Branches

| Branch | Tracks | Purpose |
|--------|--------|---------|
| `main` | Upstream bridge verbatim | Mirror of upstream. No custom changes ever. |
| `custom` | Your patches on top of `main` | All custom bridge work lives here. This is your default branch. |

## Syncing from upstream

```bash
# 1. Update main to match upstream
git checkout main
git fetch upstream
git checkout upstream/main -- whatsapp-bridge/
git commit -m "chore: sync bridge from upstream"

# 2. Rebase custom patches onto new main
git checkout custom
git rebase main

# 3. Push both
git push origin main
git push origin custom
```

If conflicts occur during rebase, git pauses for you to resolve them before continuing.