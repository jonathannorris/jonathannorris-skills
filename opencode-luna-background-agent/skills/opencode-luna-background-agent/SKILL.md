---
name: opencode-luna-background-agent
description: Delegate a task to a cheap headless OpenCode session running GPT-5.6 Luna via GitHub Copilot. Use when asked to offload, farm out, or run work on Luna or a cheaper background model
allowed-tools: Bash(opencode:*) Bash(~/.claude/scripts/luna.sh:*)
---

## When to use

- "offload this to Luna" / "run this on Luna in the background"
- "farm this out to a cheaper model"
- "kick off an opencode session to do X"

Good fits: bulk mechanical edits, repo surveys, dependency bumps, test-writing passes; work
that is verifiable after the fact. Do not delegate work that needs your own judgment mid-task,
or anything touching credentials or destructive git operations.

## Launch a session

```bash
opencode run --dir <repo-path> \
  --model github-copilot/gpt-5.6-luna --variant xhigh \
  --agent build --auto \
  "<prompt>"
```

Run it via Bash with `run_in_background: true`. The final assistant message goes to stdout;
you are re-invoked when the process exits.

Shorthand wrapper (same thing, optional): `~/.claude/scripts/luna.sh -d <repo> "<prompt>"`,
with `-v <variant>`, `-a <agent>`, `-s <session-id>`, `-j` for JSON.

## Flags that matter

- `--model github-copilot/gpt-5.6-luna` — the Copilot provider. NOT `cursor-acp/gpt-5.6-luna-*`;
  those need a separate Cursor login and fail with `Authentication required`.
- `--variant xhigh` — reasoning effort. Valid: `none|low|medium|high|xhigh|max`. On Copilot the
  effort is this flag, never part of the model id. A misspelled variant is silently ignored and
  the model runs with no reasoning at all, with no error. Verify with `-j` and check `reasoning` > 0.
- `--agent` — `build` (default, can edit), `plan`, `explore`, `general`.
- `--auto` — approve permissions that are not explicitly denied. Required for unattended runs;
  otherwise the session can hang on a permission prompt and never exit.
- `--dir` — working directory for the session. Set it explicitly; don't rely on cwd.
- `--format json` / `-j` — one JSON event per line, with per-step `tokens` and `cost`.

## Writing the prompt

The sub-agent has no access to your conversation. Include everything it needs:

- Absolute paths to the files or repo involved
- The concrete outcome, and how it should verify (the exact test or build command)
- Whether it may commit (default: no; say "do not commit, leave changes in the working tree")

## Multi-turn

Each run has a session id (in `-j` output as `sessionID`). Continue it instead of re-sending context:

```bash
opencode run --session <ses_id> --model github-copilot/gpt-5.6-luna --variant xhigh "<follow-up>"
```

## After it finishes

Never report the sub-agent's claims as your own verified result. Check the actual work:
`git -C <repo> diff --stat`, read the changed files, run the test command yourself. Luna at xhigh
is capable but will occasionally report success on work it did not finish.

## Gotchas

- Chaining setup (e.g. `chmod`) and the `--auto` run in one Bash call can trip Claude Code's
  auto-mode permission classifier. Use separate calls.
- Tool calls outside the paths allowed in `~/.config/opencode/opencode.json`
  (`permission.external_directory`, currently `~/git/**` and `~/.config/**`) may prompt. With
  `--auto` they are approved; without it the run can stall.
- The `cost` in JSON output is a per-token estimate. Copilot bills by premium request, so it does
  not reflect actual spend; treat it as relative signal only.
- Startup is ~1-2s per invocation. For many sequential prompts, reuse a session or run
  `opencode serve` and attach with `--attach`.
