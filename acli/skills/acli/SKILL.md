---
name: acli
description: Interact with Jira using the Atlassian CLI (acli). Covers looking up tickets, searching with JQL, creating and editing work items, transitioning status, adding comments, and parsing JSON output. Use when asked to look up, create, edit, transition, or comment on Jira tickets — e.g. "look up ICP-1234", "create a ticket for X", "transition ICP-5678 to In Progress", "what's assigned to me in the current sprint?".
---

## Which tool to use

At Dynatrace, **always prefer `acli-pii` over plain `acli`**. `acli-pii` mirrors the full `acli` command surface but routes all traffic through a PII-redacting proxy that automatically pseudonymises personal data before responses reach the terminal.

```
acli-pii  →  Dynatrace PII-redacting proxy  →  Atlassian Cloud APIs
```

Use plain `acli` only when `acli-pii` is unavailable or you are explicitly working outside the Dynatrace Jira instance.

For setup, authentication, and PII control flags (`--pii-skip`, `--pii-only`), see [REFERENCE.md](REFERENCE.md).

---

## Quick command reference

All examples below use `acli-pii`. Replace with `acli` if using the plain CLI.

### View a ticket

```bash
acli-pii jira workitem view ICP-123
acli-pii jira workitem view ICP-123 --fields "summary,status,assignee,description"
acli-pii jira workitem view ICP-123 --json
```

### Search (JQL)

```bash
acli-pii jira workitem search --jql "assignee = currentUser() AND status != Done"
acli-pii jira workitem search --jql "project = ICP AND sprint in openSprints()" --paginate
acli-pii jira workitem search --jql "project = ICP" --count
acli-pii jira workitem search --jql "project = ICP" --fields "key,summary,status,priority"
```

### Create

```bash
acli-pii jira workitem create --project ICP --type Story --summary "Title here"
acli-pii jira workitem create --project ICP --type Bug --summary "Title" \
  --description "Details" --field priority=High --field labels=bug
```

### Edit

```bash
acli-pii jira workitem edit --key ICP-123 --summary "Updated title" --yes
acli-pii jira workitem edit --key ICP-123 --field priority=Critical --yes
```

### Transition

```bash
acli-pii jira workitem transition --key ICP-123 --status "In Progress" --yes
acli-pii jira workitem transition --key ICP-123 --status "Done" --yes
```

### Assign

```bash
acli-pii jira workitem assign --key ICP-123 --assignee @me
acli-pii jira workitem assign --key ICP-123 --assignee user@dynatrace.com
```

### Comment

```bash
acli-pii jira workitem comment create --key ICP-123 --body "Comment text"
acli-pii jira workitem comment list --key ICP-123
```

---

## Formatting descriptions and comments

`--description` / `--body` take **either** plain text **or** Atlassian Document Format (ADF) JSON. Plain text is stored as plain paragraphs: a leading `-` stays a literal dash and `###` stays literal `###`. You never get real bullet lists, headings, or inline code from plain text.

Pick based on structure:

| Content | Use |
|---|---|
| A sentence or two, no structure | `--description "..."` |
| Headings, bullet lists, inline code, links | ADF via `--description-file` |

```bash
# Plain text: fine for short, unstructured content
acli-pii jira workitem edit --key ICP-123 --description "Bumped the timeout to 4s." --yes

# ADF: anything with real structure
acli-pii jira workitem edit --key ICP-123 --description-file desc.json --yes
acli-pii jira workitem create --project ICP --type Story --summary "Title" --description-file desc.json
acli-pii jira workitem comment create --key ICP-123 --body-file comment.json
```

Always validate the JSON before sending; `acli` reports a generic failure on a malformed document.

```bash
python3 -c "import json;json.load(open('desc.json'))" && acli-pii jira workitem edit --key ICP-123 --description-file desc.json --yes
```

Then read it back, because a silently mangled description looks identical to a successful edit:

```bash
acli-pii jira workitem view ICP-123 --fields description
```

Note that `view` flattens headings into bare text, so use it to confirm the words are present, not to judge the rendering. Check the Jira UI for that.

For the ADF node reference and a copy-paste template, see [REFERENCE.md](REFERENCE.md).

---

## Common JQL patterns

```bash
# My open work
"assignee = currentUser() AND status != Done"

# Current sprint
"project = ICP AND sprint in openSprints()"

# Recent updates
"project = ICP AND updated >= -7d ORDER BY updated DESC"

# Text search
"project = ICP AND text ~ 'feature flag'"

# Bugs only
"project = ICP AND issuetype = Bug AND status != Done"
```


For JSON scripting with `jq` and CSV export, see [REFERENCE.md](REFERENCE.md).
