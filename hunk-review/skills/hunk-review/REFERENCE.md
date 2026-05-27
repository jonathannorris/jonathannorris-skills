## New files in working-tree reviews

`hunk diff` includes untracked files by default. To show tracked changes only:

```bash
hunk session reload --repo . -- diff --exclude-untracked
```

## Common errors

- **"No visible diff file matches ..."** -- the file is not in the loaded review. Check `context`, then `reload` if needed.
- **"No active Hunk sessions"** -- ask the user to open Hunk in their terminal.
- **"Multiple active sessions match"** -- pass `<session-id>` explicitly.
- **"No active Hunk session matches session path ..."** -- for advanced split-path reloads, verify the live window `Path` via `hunk session get` or `list`, then use `--session-path`.
- **"Pass the replacement Hunk command after `--`"** -- include `--` before the nested `diff` / `show` command.
- **"Pass --stdin to read batch comments from stdin JSON."** -- `comment apply` only reads its batch payload from stdin.
- **"Specify exactly one navigation target"** -- pick one of `--hunk`, `--old-line`, or `--new-line`.
- **"Specify either --next-comment or --prev-comment, not both."** -- choose one comment-navigation direction.
