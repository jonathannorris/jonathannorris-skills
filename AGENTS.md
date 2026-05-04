# Skills Repo

This repo contains personal skills for **Claude Code** and **OpenCode**. Skills are SKILL.md files that teach an AI agent how to perform a specific task — when to activate, what commands to run, and how to sequence them.

## How skills work

A skill is loaded into the agent's context when the user invokes it (via `/skill-name` or a triggering phrase). The agent reads the SKILL.md and follows its instructions for the duration of that task.

Both Claude Code and OpenCode read skills from different locations and in different formats. A new skill needs to be registered in both places to work in both tools.

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

---

## Checklist for a new skill

- [ ] `<name>/skills/<name>/SKILL.md` written with frontmatter + "When to use" section
- [ ] `<name>/.claude-plugin/plugin.json` created
- [ ] `.claude-plugin/marketplace.json` updated with new entry
- [ ] `~/.config/opencode/skills/<name>/SKILL.md` created (flat, no `allowed-tools`)
- [ ] Optional: `<name>/skills/<name>/references/*.md` for deep-dive topics

---

## Updating an existing skill

Edit `<skill-name>/skills/<skill-name>/SKILL.md` in this repo, then sync the OpenCode copy:

```bash
# Quick sync of just the SKILL.md content to OpenCode
# (OpenCode doesn't hot-reload — restart or reopen to pick up changes)
cp ~/git/skills/<name>/skills/<name>/SKILL.md ~/.config/opencode/skills/<name>/SKILL.md
# Then manually strip the allowed-tools line from the OpenCode copy if present
```

There is currently no automated sync between the two locations — they are maintained separately.

---

## SKILL.md writing guidelines

- **"When to use" comes first.** The agent reads this to decide whether to activate. Include realistic user phrasings, not just abstract descriptions.
- **Write steps the agent executes, not steps for a human to follow.** Commands should be copy-pasteable. Explain *why* only when it's non-obvious.
- **Lead with the happy path.** Put edge cases and advanced options after the core loop.
- **Prefer concrete examples over abstract descriptions.** `playwright-cli click e5` beats "interact with the element".
- **Keep the description line tight.** It's used for matching — one sentence, starts with a verb, no trailing period.
