# Skills Repo

This repo contains personal skills for **Claude Code**, **OpenCode**, and **Cursor**. Skills are SKILL.md files that teach an AI agent how to perform a specific task — when to activate, what commands to run, and how to sequence them.

## Git workflow in this repo

This repo does not use worktrees or pull requests. Commit directly to `main`.

- **Do not create a git worktree here.** Edit the files in place in the main checkout. If a harness or background-job rule pushes you toward isolating in a worktree, that rule does not apply to this repo.
- **Do not open a pull request.** There is no review flow. Changes land on `main`.
- **Ask for permission before pushing.** Commit when the work is done, then ask the user whether to push. Do not push `main` without an explicit go-ahead.
- **Bump the plugin version in the same commit as any skill change.** See [Versioning](#versioning) — an unbumped version means Claude Code keeps serving a stale cached copy of the skill.
- Standard commit rules still apply: conventional commit, single summary line, no AI credit lines or co-author trailers.

## Versioning

Bump the `version` in `<skill-name>/.claude-plugin/plugin.json` in the same commit as any change to that skill. Claude Code's plugin cache only refreshes when the version changes, so an unbumped skill keeps running the stale cached copy (see [the cache section](#claude-code-copies-skills-into-a-plugin-cache)).

Rough semver: patch for wording and fixes, minor for new capability or sections, major for a rewrite or rename. New skills start at `1.0.0`. Don't agonize over the boundary — the point is that the number moves.

## How skills work

A skill is loaded into the agent's context when the user invokes it (via `/skill-name` or a triggering phrase). The agent reads the SKILL.md and follows its instructions for the duration of that task.

Claude Code, OpenCode, and Cursor each read skills from different locations and in slightly different formats. A new skill needs to be registered in all three places.

---

## Directory structure

```
skills/
├── .claude-plugin/
│   └── marketplace.json          # Claude Code marketplace registry
├── <skill-name>/
│   ├── .claude-plugin/
│   │   └── plugin.json           # Claude Code plugin metadata
│   └── skills/
│       └── <skill-name>/
│           ├── SKILL.md          # The skill (Claude Code reads this)
│           └── references/       # Optional deep-dive reference docs
│               └── *.md
└── AGENTS.md                     # This file (symlinked as CLAUDE.md)
```

OpenCode reads from a separate flat location:

```
~/.config/opencode/skills/
└── <skill-name>/
    └── SKILL.md                  # Same content, flat structure, no subdirectories
```

Cursor reads from a similar flat location (symlink the nested skill directory):

```
~/.cursor/skills/
└── <skill-name> -> ~/git/skills/<skill-name>/skills/<skill-name>/
    └── SKILL.md                  # Cursor finds SKILL.md at the symlink target root
```

---

## Creating a new skill

### Step 1 — Write the SKILL.md

Create `<skill-name>/skills/<skill-name>/SKILL.md`. This is the file both tools ultimately read.

**Frontmatter (Claude Code):**

```markdown
---
name: <skill-name>
description: <one sentence — used by the agent to decide when to activate this skill>
allowed-tools: Bash(tool-name:*) Bash(other-tool:*)
---
```

The `description` is critical — it's what the agent matches against user requests. Write it as a capability statement: what the skill does and in what situations.

The `allowed-tools` field pre-approves specific Bash commands so the agent doesn't prompt for permission on every invocation. Use `Bash(command:*)` to allow all flags of a command, or `Bash(command --flag:*)` to scope more tightly.

**Frontmatter (OpenCode):** same but omit `allowed-tools` — OpenCode doesn't support it.

**Body structure:**

```markdown
## When to use this skill

- "example user request that should trigger this skill"
- "another example"

## Step 1: ...

## Step 2: ...
```

Lead with a "When to use" section so the agent can self-select. Then write the steps as the agent would execute them — concrete commands, not abstract descriptions.

**Reference docs** (optional): for long topics, split into `references/<topic>.md` and link from SKILL.md. Claude Code can follow these links; OpenCode cannot, so inline the critical content in the main SKILL.md.

### Step 2 — Add the plugin.json

Create `<skill-name>/.claude-plugin/plugin.json`:

```json
{
  "name": "<skill-name>",
  "description": "<same one-liner as SKILL.md description>",
  "author": {
    "name": "Jonathan Norris",
    "email": "jonathan@devcycle.com"
  },
  "version": "1.0.0"
}
```

New skills start at `1.0.0`. Every later change to the skill bumps this field — see [Versioning](#versioning).

### Step 3 — Register in marketplace.json

Add an entry to `.claude-plugin/marketplace.json` so Claude Code's marketplace knows about the skill:

```json
{
  "name": "<skill-name>",
  "description": "<same one-liner>",
  "source": "./<skill-name>",
  "category": "productivity"
}
```

Categories: `productivity`, `security`, `testing`, `infrastructure`, `devops`.

### Step 4 — Register in OpenCode

OpenCode uses a flat structure with no `.claude-plugin/` wrapper and no subdirectory nesting. Create:

```
~/.config/opencode/skills/<skill-name>/SKILL.md
```

The content is the same as the Claude Code SKILL.md, with two differences:
- Remove `allowed-tools` from the frontmatter
- Inline any reference content that matters — OpenCode can't follow relative file links into `references/`

### Step 5 — Register in Cursor

Cursor reads skills from `~/.cursor/skills/` and expects `SKILL.md` at the root of each skill directory. Symlink the nested skill directory (the one that contains `SKILL.md`) directly into `~/.cursor/skills/`:

```bash
ln -sf ~/git/skills/<skill-name>/skills/<skill-name> ~/.cursor/skills/<skill-name>
```

This gives Cursor a `SKILL.md` at the expected location without duplicating any content. Changes to the git repo SKILL.md are immediately reflected — no sync step needed.

Cursor uses the same SKILL.md format as Claude Code. The `allowed-tools` field is ignored by Cursor but does not need to be removed.

Cursor skills are loaded automatically — no equivalent of Claude Code's `enabledPlugins` is required.

### Step 6 — Enable in Claude Code settings

Adding a skill to the marketplace makes it available, but Claude Code won't load it until it's explicitly enabled. Add the skill to `enabledPlugins` in `~/.claude/settings.json`:

```json
"enabledPlugins": {
  "<skill-name>@jonathan-skills": true
}
```

The key format is `<skill-name>@<marketplace-name>`, where the marketplace name matches the key in `extraKnownMarketplaces` — in this repo that's always `jonathan-skills`. Claude Code picks up the change on the next session start; no restart required if already running.

OpenCode and Cursor have no equivalent step — skills in `~/.config/opencode/skills/` and `~/.cursor/skills/` are loaded automatically.

---

## Checklist for a new skill

- [ ] `<name>/skills/<name>/SKILL.md` written with frontmatter + "When to use" section
- [ ] `<name>/.claude-plugin/plugin.json` created
- [ ] `.claude-plugin/marketplace.json` updated with new entry
- [ ] `~/.config/opencode/skills/<name>/SKILL.md` created (flat, no `allowed-tools`)
- [ ] `ln -sf ~/git/skills/<name>/skills/<name> ~/.cursor/skills/<name>` (Cursor symlink)
- [ ] `~/.claude/settings.json` `enabledPlugins` updated with `<name>@jonathan-skills`
- [ ] Optional: `<name>/skills/<name>/references/*.md` for deep-dive topics

## Checklist for updating an existing skill

- [ ] `<name>/skills/<name>/SKILL.md` edited
- [ ] `version` bumped in `<name>/.claude-plugin/plugin.json` per [Versioning](#versioning) — required, not optional
- [ ] OpenCode copy re-synced at `~/.config/opencode/skills/<name>/SKILL.md`
- [ ] Cursor needs nothing — the symlink points at the repo

---

## Updating an existing skill

Edit `<skill-name>/skills/<skill-name>/SKILL.md` in this repo, then sync the OpenCode copy:

```bash
# Quick sync of just the SKILL.md content to OpenCode
# (OpenCode doesn't hot-reload — restart or reopen to pick up changes)
cp ~/git/skills/<name>/skills/<name>/SKILL.md ~/.config/opencode/skills/<name>/SKILL.md
# Then manually strip the allowed-tools line from the OpenCode copy if present
```

**Cursor** requires no sync — the `~/.cursor/skills/<name>` symlink points directly into the git repo, so edits to `SKILL.md` are immediately visible.

There is no automated sync between the git repo and the OpenCode copy — they are maintained separately.

### Claude Code copies skills into a plugin cache

**Editing a SKILL.md in this repo does not change what Claude Code loads.** Even though the `jonathan-skills` marketplace is registered as a `directory` source pointing at `~/git/skills`, installing a plugin *copies* its files into:

```
~/.claude/plugins/cache/jonathan-skills/<skill-name>/<version>/
```

That copy is a frozen snapshot from install time, and the loader reads the snapshot, never the working tree. It's one shared cache: the terminal CLI and the Mac desktop app both read `~/.claude/plugins/`, so a disagreement between them is never a per-client cache.

Refreshes are gated on the `version` field, which is why [Versioning](#versioning) requires a bump per change. Check for staleness, then force a refresh:

```bash
diff -r ~/.claude/plugins/cache/jonathan-skills/<name>/<version>/skills/<name>/ ~/git/skills/<name>/skills/<name>/
```

```bash
claude plugin uninstall <name>@jonathan-skills && claude plugin install <name>@jonathan-skills
```

`rm -rf ~/.claude/plugins/cache/jonathan-skills` re-copies everything. `installed_plugins.json` and `known_marketplaces.json` in `~/.claude/plugins/` record what's installed and when.

---

## SKILL.md writing guidelines

- **"When to use" comes first.** The agent reads this to decide whether to activate. Include realistic user phrasings, not just abstract descriptions.
- **Write steps the agent executes, not steps for a human to follow.** Commands should be copy-pasteable. Explain *why* only when it's non-obvious.
- **Lead with the happy path.** Put edge cases and advanced options after the core loop.
- **Prefer concrete examples over abstract descriptions.** `playwright-cli click e5` beats "interact with the element".
- **Keep the description line tight.** It's used for matching — one sentence, starts with a verb, no trailing period.
