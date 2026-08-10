---
name: opencode-luna-background-agent
description: Delegate a task to a cheap headless OpenCode session running GPT-5.6 Luna via GitHub Copilot. Use when asked to offload, farm out, or run work on Luna or a cheaper background model
allowed-tools: Bash(opencode:*) Bash(~/.claude/scripts/luna.sh:*)
---

## When to use

- "offload this to Luna" / "run this on Luna in the background"
- "farm this out to a cheaper model"
- "kick off an opencode session to do X"

Good fits: bulk mechanical edits, repo surveys, dependency bumps, test-writing passes; work that
is verifiable after the fact. Don't delegate work needing your judgment mid-task, or anything
touching credentials or destructive git operations.

## Launch

```bash
opencode run --dir <repo-path> \
  --model github-copilot/gpt-5.6-luna --variant xhigh \
  --agent build --auto \
  "<prompt>"
```

Run via Bash with `run_in_background: true`. The final assistant message goes to stdout; you are
re-invoked when the process exits. Shorthand: `~/.claude/scripts/luna.sh -d <repo> "<prompt>"`
(`-v` variant, `-a` agent, `-s` session id, `-j` JSON events).

Each of the four flags above prevents a specific failure:

- **`--model github-copilot/gpt-5.6-luna`** — NOT `cursor-acp/gpt-5.6-luna-*`. Those need a
  separate Cursor login and fail with `Authentication required`.
- **`--variant xhigh`** — reasoning effort (`none|low|medium|high|xhigh|max`). On Copilot this is
  a flag, never part of the model id. A misspelled variant is silently ignored and the model runs
  with no reasoning and no error; confirm with `-j` and check `reasoning` > 0.
- **`--auto`** — approves permissions not explicitly denied. Without it an unattended run can hang
  on a prompt and never exit.
- **`--dir`** — set the working directory explicitly; don't rely on cwd.

Agents: `build` (default, can edit), `plan`, `explore`, `general`. To continue a session instead of
re-sending context, pass `--session <ses_id>` (the id is in `-j` output as `sessionID`).

## Write a self-contained prompt

The sub-agent sees none of your conversation. Include absolute paths, the concrete outcome, the
exact verification command, and whether it may commit (default: "do not commit, leave changes in
the working tree").

## Verify before reporting

Never pass the sub-agent's claims off as your own verified result. Run `git -C <repo> diff --stat`,
read the changed files, and run the test command yourself. Luna at xhigh is capable but will
occasionally report success on work it did not finish.
