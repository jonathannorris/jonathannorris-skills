---
name: acli
description: Interact with Jira using the Atlassian CLI (acli). Covers looking up tickets, searching with JQL, creating and editing work items, transitioning status, adding comments, and parsing JSON output. Use when asked to look up, create, edit, transition, or comment on Jira tickets — e.g. "look up ICP-1234", "create a ticket for X", "transition ICP-5678 to In Progress", "what's assigned to me in the current sprint?".
---

## Keeping personal data out of context

`acli` talks directly to Atlassian, so responses include real names and emails. Request only the fields the task needs (for example `--fields "summary,status,description"`) and avoid pulling assignee, reporter, or watcher data unless asked.

For setup and authentication, see [REFERENCE.md](REFERENCE.md).

---

## Quick command reference

### View a ticket

```bash
acli jira workitem view ICP-123
acli jira workitem view ICP-123 --fields "summary,status,assignee,description"
acli jira workitem view ICP-123 --json
```

### Search (JQL)

```bash
acli jira workitem search --jql "assignee = currentUser() AND status != Done"
acli jira workitem search --jql "project = ICP AND sprint in openSprints()" --paginate
acli jira workitem search --jql "project = ICP" --count
acli jira workitem search --jql "project = ICP" --fields "key,summary,status,priority"
```

### Create

```bash
acli jira workitem create --project ICP --type Story --summary "Title here"
acli jira workitem create --project ICP --type Bug --summary "Title" \
  --description "Details" --field priority=High --field labels=bug
```

### Edit

```bash
acli jira workitem edit --key ICP-123 --summary "Updated title" --yes
acli jira workitem edit --key ICP-123 --field priority=Critical --yes
```

### Transition

```bash
acli jira workitem transition --key ICP-123 --status "In Progress" --yes
acli jira workitem transition --key ICP-123 --status "Done" --yes
```

### Assign

```bash
acli jira workitem assign --key ICP-123 --assignee @me
acli jira workitem assign --key ICP-123 --assignee user@dynatrace.com
```

### Comment

```bash
acli jira workitem comment create --key ICP-123 --body "Comment text"
acli jira workitem comment create --key ICP-123 --body-file comment.json   # ADF, see below
acli jira workitem comment create --key ICP-123 --body-file comment.json --edit-last
acli jira workitem comment list --key ICP-123
```

`--edit-last` replaces your most recent comment on the item instead of adding a new one. Use it to fix a comment you already posted (wording or formatting) so the ticket doesn't end up with a duplicate.

---

## Formatting descriptions and comments

`--description` / `--body` take **either** plain text **or** Atlassian Document Format (ADF) JSON. Plain text is stored as plain paragraphs: a leading `-` stays a literal dash and `###` stays literal `###`. You never get real bullet lists, headings, or inline code from plain text.

Pick based on structure:

| Content | Use |
|---|---|
| A sentence or two, no structure | `--description "..."` / `--body "..."` |
| Headings, bullet lists, inline code, links | ADF via `--description-file` / `--body-file` |

If a draft has lines starting with `-`, `*`, `#`, or contains identifiers, file paths, or regexes that belong in backticks, it needs ADF. Posting it as plain text shows the raw dashes and lets long regexes wrap mid-pattern. Wrap field names, schema IDs, paths, patterns, and commands in `code` marks.

Build ADF with a small Python script (helpers plus `json.dump`) rather than hand-writing JSON: it keeps backslashes in regexes and quotes correct without manual escaping. See the builder in [REFERENCE.md](REFERENCE.md).

```bash
# Plain text: fine for short, unstructured content
acli jira workitem edit --key ICP-123 --description "Bumped the timeout to 4s." --yes

# ADF: anything with real structure
acli jira workitem edit --key ICP-123 --description-file desc.json --yes
acli jira workitem create --project ICP --type Story --summary "Title" --description-file desc.json
acli jira workitem comment create --key ICP-123 --body-file comment.json
```

Always validate the JSON before sending; `acli` reports a generic failure on a malformed document.

```bash
python3 -c "import json;json.load(open('desc.json'))" && acli jira workitem edit --key ICP-123 --description-file desc.json --yes
```

Then read it back, because a silently mangled description looks identical to a successful edit:

```bash
acli jira workitem view ICP-123 --fields description
```

For comments, read back with `acli jira workitem comment list --key ICP-123 --json`. If the body comes back as literal JSON text (`{"type":"doc"...`), acli stored the file as plain text: fix the JSON and re-post with `--edit-last`.

Note that `view` and `comment list` flatten headings and code marks into bare text, so use them to confirm the words are present, not to judge the rendering. Check the Jira UI for that.

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
