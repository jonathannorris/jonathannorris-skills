---
name: acli
description: Interact with Jira using the Atlassian CLI (acli). Covers looking up tickets, searching with JQL, creating and editing work items, transitioning status, adding comments, and parsing JSON output.
---

## When to use this skill

Use this skill when asked to interact with Jira from the terminal. Common triggers:
- "look up ticket ICP-1234"
- "create a Jira ticket for..."
- "find all open bugs in project X"
- "transition ICP-5678 to In Progress"
- "add a comment to ICP-999"
- "what's assigned to me in the current sprint?"
- "update the description on ICP-100"

## Which tool to use

At Dynatrace, **always prefer `acli-pii` over plain `acli`**. `acli-pii` mirrors the full `acli` command surface but routes all traffic through a PII-redacting proxy that automatically pseudonymises personal data (names, emails, etc.) before responses reach the terminal.

```
acli-pii  →  Dynatrace PII-redacting proxy  →  Atlassian Cloud APIs
```

Use plain `acli` only when `acli-pii` is unavailable or you are explicitly working outside the Dynatrace Jira instance.

For full `acli-pii` command docs, PII control flags (`--pii-skip`, `--pii-only`), Confluence support, and field coercion details, see the `dt-atlassian-pii` skill at:

```
~/git/Dynatrace/feature-managment-app/.claude/skills/dt-atlassian-pii/SKILL.md
```

---

## Setup

### Binary

`acli-pii` is installed at `~/.local/bin/acli-pii` (already on `PATH`). Verify with:

```bash
acli-pii version
```

If missing, copy the macOS binary from the bundled utils and make it executable:

```bash
cp ~/git/Dynatrace/feature-managment-app/.claude/skills/dt-atlassian-pii/utils/dt-acli-pii-sanitize/acli-pii-darwin \
   ~/.local/bin/acli-pii
chmod +x ~/.local/bin/acli-pii
```

### Config

`~/.acli-pii/config.yaml` is pre-configured:

```yaml
site: "dt-rnd.atlassian.langdock.internal.dynatrace.com"
email: "jonathan.norris@dynatrace.com"
```

### Credentials (env vars)

The following are set in `~/.zshrc`:

```bash
ACLI_JIRA_TOKEN   # Atlassian API token
ACLI_JIRA_SITE    # dt-rnd.atlassian.net
ACLI_JIRA_EMAIL   # jonathan.norris@dynatrace.com
```

### Authentication

```bash
# acli-pii (preferred — PII-safe)
echo "$ACLI_JIRA_TOKEN" | acli-pii jira auth login \
  --site "dt-rnd.atlassian.langdock.internal.dynatrace.com" \
  --email "$ACLI_JIRA_EMAIL" \
  --token

# plain acli (fallback)
echo "$ACLI_JIRA_TOKEN" | acli jira auth login \
  --site "$ACLI_JIRA_SITE" \
  --email "$ACLI_JIRA_EMAIL" \
  --token
```

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

---

## Scripting with JSON + jq

```bash
# Extract a single field
acli-pii jira workitem view ICP-123 --json | jq '.fields.status.name'

# Get all keys from a search
acli-pii jira workitem search --jql "project = ICP AND status = 'To Do'" --json \
  | jq -r '.[].key'

# Export to CSV
acli-pii jira workitem search --jql "project = ICP" --csv > results.csv
```

---

## Key differences: `acli-pii` vs `acli`

| | `acli-pii` | `acli` |
|---|---|---|
| Traffic routing | Dynatrace PII proxy | Direct to Atlassian |
| PII in output | Pseudonymised (`<PERSON_1>`, `<EMAIL_1>`) | Real values |
| Extra flags | `--pii-skip`, `--pii-only` | Not available |
| Confluence support | Yes | Yes |
| Config location | `~/.acli-pii/config.yaml` | OS keyring |
| Proxy site | `dt-rnd.atlassian.langdock.internal.dynatrace.com` | `dt-rnd.atlassian.net` |
