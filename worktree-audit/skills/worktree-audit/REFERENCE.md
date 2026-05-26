# Worktree Audit Reference

## Classification rules

Apply in order. First matching rule wins.

| Condition | Classification |
|---|---|
| Worktree directory missing (gitdir stale) | **Prune** -- auto via `worktree prune` |
| Has uncommitted changes | **Keep** |
| Remote branch exists, last commit within 7 days | **Keep** |
| Remote branch deleted, no uncommitted changes | **Prune** |
| Detached HEAD reachable from default branch, no uncommitted changes | **Prune** |
| No remote branch, no uncommitted changes, last commit >14 days ago | **Prune** |
| Remote branch exists, last commit >7 days old | **Investigate** -- may be an open PR |
| No remote branch, has uncommitted changes, >14 days old | **Investigate** -- review changes |
| Anything unclear | **Investigate** |

## Detecting detached HEAD

A worktree in detached HEAD state has no `branch` line in `--porcelain` output. To check if the commit is reachable from the default branch:

```bash
git -C <repo> merge-base --is-ancestor <commit> origin/main && echo "reachable" || echo "not reachable"
```

If reachable and no uncommitted changes: prune.

## Porcelain output format

```
worktree /path/to/worktree
HEAD abc123def
branch refs/heads/my-branch

worktree /path/to/another
HEAD def456abc
detached
```

Missing `branch` line = detached HEAD.

## External tool worktrees

These tools create worktrees in non-standard locations. They are valid git worktrees and must be removed via `git worktree remove`, not by deleting the directory directly.

| Tool | Path pattern |
|---|---|
| Cursor | `~/.cursor/worktrees/<repo>/<id>` |
| OpenCode | `~/.local/share/opencode/worktree/<hash>/<name>` |
| Superconductor | `~/.superconductor/worktrees/<repo>/<name>` |
