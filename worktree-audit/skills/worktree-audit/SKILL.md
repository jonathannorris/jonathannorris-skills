---
name: worktree-audit
description: Audit and prune stale git worktrees across all local repos. Use when asked to review, clean up, or prune worktrees, or to find which worktrees can be removed.
---

## When to use

- "review all the open git worktrees"
- "prune stale worktrees" / "clean up old worktrees"
- "which worktrees can I get rid of"

## Workflow

### 1. Scan all repos

```bash
find ~/git/DevCycle ~/git/OpenFeature ~/git/Dynatrace ~/git/DevCycle-Labs ~/git/TestRepos \
  -maxdepth 1 -mindepth 1 -type d 2>/dev/null | sort
```

Verify each is a git repo: `git -C <dir> rev-parse --git-dir 2>/dev/null`. Skip non-repos.

### 2. List and analyze worktrees

For each repo, run these together:

```bash
git -C <repo> worktree list --porcelain       # all worktrees
git -C <repo> worktree prune --dry-run -v     # already-stale entries
```

Skip the first entry (main worktree). For each additional entry, collect:

```bash
git -C <repo> ls-remote --heads origin <branch>          # remote exists?
git -C <worktree-path> status --short 2>/dev/null         # uncommitted changes?
git -C <repo> log -1 --format="%ci" <branch> 2>/dev/null # last commit date
```

See [REFERENCE.md](REFERENCE.md) for classification rules (prune / keep / investigate).

### 3. Present report

Three tables: **Safe to prune**, **Keep**, **Investigate**.
Columns: Repo | Worktree path | Branch | Last commit | Reason.

**Do not run any cleanup until the user confirms.**

### 4. Clean up (after confirmation)

```bash
# Auto-prune entries whose directory is already gone
git -C <repo> worktree prune -v

# Manual remove
git -C <repo> worktree remove <worktree-path>
```

Run in batches by repo. Only use `--force` if the directory is confirmed missing.

### 5. Summary

Report: how many pruned, how many kept, what remains in Investigate.

## Key rules

- Never remove a worktree with uncommitted changes without explicit user instruction
- The main worktree (first in `git worktree list`) is never prunable
- `git worktree remove` removes the working tree only; branches are unaffected
- Worktrees from external tools (Cursor, OpenCode, Superconductor) under `~/.cursor/worktrees/`, `~/.local/share/opencode/worktree/`, `~/.superconductor/worktrees/` are valid; confirm with user before removing
- For repos where the main checkout's own branch has a deleted remote, note it as a separate concern
