---
name: worktree-audit
description: Audit and prune git worktrees across all local repos. Scans repos under ~/git, identifies stale/prunable worktrees (deleted remote branch, no uncommitted changes), and performs cleanup with user confirmation.
---

## When to use this skill

Use this skill when asked to audit, review, clean up, or prune git worktrees. This includes requests like:
- "review all the open git worktrees"
- "prune stale worktrees"
- "clean up old worktrees"
- "which worktrees can I get rid of"

## Step 1: Find all repos

Scan all repo directories. The repos live under these top-level directories:

```bash
find ~/git/DevCycle ~/git/OpenFeature ~/git/Dynatrace ~/git/DevCycle-Labs ~/git/TestRepos \
  -maxdepth 1 -mindepth 1 -type d 2>/dev/null | sort
```

For each directory, check if it is a git repo:

```bash
git -C <dir> rev-parse --git-dir 2>/dev/null
```

Skip directories that are not git repos.

## Step 2: List worktrees for each repo

For every repo, list its worktrees in porcelain format:

```bash
git -C <repo> worktree list --porcelain
```

The porcelain output looks like:

```
worktree /path/to/worktree
HEAD abc123def
branch refs/heads/my-branch

worktree /path/to/another
HEAD def456abc
detached
```

Skip the first entry (the main worktree). For all additional entries, record:
- Path
- HEAD commit hash
- Branch name (or "detached")

Also run the dry-run prune to surface entries git has already detected as stale:

```bash
git -C <repo> worktree prune --dry-run -v 2>&1
```

Any output here means the worktree's `.git` metadata is gone and it can be pruned immediately.

## Step 3: Analyze each worktree

For each non-main worktree, gather:

### Does the remote branch still exist?

```bash
git -C <repo> ls-remote --heads origin <branch-name>
# Empty output = branch deleted on remote
# Non-empty = branch exists
```

For detached HEAD worktrees, check whether the commit is reachable from the default branch:

```bash
git -C <repo> merge-base --is-ancestor <commit> origin/main && echo "reachable" || echo "not reachable"
```

### Does the worktree directory still exist on disk?

```bash
test -d <worktree-path> && echo "exists" || echo "missing"
```

Missing directories are auto-detected by `worktree prune`; they don't need manual removal.

### Are there uncommitted changes?

```bash
git -C <worktree-path> status --short 2>/dev/null
```

If the path doesn't exist, skip this check.

### When was the last commit on the branch?

```bash
git -C <repo> log -1 --format="%ci" <branch-name> 2>/dev/null
# Or for detached HEAD:
git -C <repo> log -1 --format="%ci" <commit> 2>/dev/null
```

## Step 4: Classify each worktree

Apply these rules in order:

| Condition | Classification |
|---|---|
| Worktree directory missing (gitdir stale) | **Prune** (auto-prune via `worktree prune`) |
| Has uncommitted changes | **Keep** |
| Remote branch exists AND last commit within 7 days | **Keep** |
| Remote branch deleted AND no uncommitted changes | **Prune** |
| Detached HEAD at commit reachable from default branch, no uncommitted changes | **Prune** |
| Remote branch deleted, last commit >30 days ago, no uncommitted changes | **Prune** |
| Remote branch exists, last commit >7 days old | **Investigate** (may be an open PR; check) |
| No remote branch, has uncommitted changes | **Keep** (or **Investigate** if >14 days old) |
| No remote branch, no uncommitted changes, last commit >14 days ago | **Prune** |
| Anything unclear | **Investigate** |

## Step 5: Present the report

Group findings into three sections and present them as tables:

### Safe to prune
Worktrees with no uncommitted changes and no active remote branch. Show:
- Repo
- Worktree path (abbreviated)
- Branch
- Last commit date
- Reason (e.g., "remote deleted", "detached at main", "2 months old, no remote")

### Keep
Worktrees that are clearly active. No action needed; just list them briefly.

### Investigate
Worktrees with ambiguous status. For each, state exactly what needs to be confirmed before pruning (e.g., "check if PR was submitted", "has uncommitted changes -- review before removing").

## Step 6: Prune auto-detected stale entries

For repos where `worktree prune --dry-run` produced output, run the actual prune:

```bash
git -C <repo> worktree prune -v
```

This is safe for any entry whose directory is already gone. Do this first, before manual removes, since it handles the easy cases.

IMPORTANT: Always present the full report to the user before pruning anything. Do not run any cleanup commands until the user confirms.

## Step 7: Remove safe worktrees

After the user confirms, remove worktrees classified as safe to prune:

```bash
git -C <repo> worktree remove <worktree-path>
```

If the worktree directory no longer exists on disk but wasn't caught by `worktree prune` (rare), use the `--force` flag:

```bash
git -C <repo> worktree remove --force <worktree-path>
```

Run these in batches by repo. After each batch, confirm the repo's worktree list is as expected:

```bash
git -C <repo> worktree list
```

Do NOT delete branches. `git worktree remove` only removes the working tree; the branch itself stays and can still be checked out or deleted separately if desired.

## Step 8: Summary

After cleanup, report:
- How many worktrees were pruned
- How many were kept
- Which worktrees still need the user's attention (the Investigate list)

## Important reminders

- **Never remove a worktree with uncommitted changes** without explicit user instruction. Always check `git status --short` first.
- Worktrees created by external tools (Cursor, OpenCode, Superconductor, etc.) under `~/.cursor/worktrees/`, `~/.local/share/opencode/worktree/`, `~/.superconductor/worktrees/` are valid worktrees tracked by the repo. They are safe to remove when the tool is no longer using them, but confirm with the user first.
- The main worktree (first entry in `git worktree list`) is never prunable.
- `git worktree prune` only removes the metadata entry; it does not delete branches.
- `git worktree remove` removes both the working tree directory and the metadata entry.
- Check `git worktree list --porcelain` output carefully: a missing `branch` line means detached HEAD.
- For repos where the main checkout's own branch has a deleted remote, note this as a separate concern -- the repo's HEAD state may need attention.
