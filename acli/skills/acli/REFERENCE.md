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

## `acli-pii` vs `acli`

| | `acli-pii` | `acli` |
|---|---|---|
| Traffic routing | Dynatrace PII proxy | Direct to Atlassian |
| PII in output | Pseudonymised (`<PERSON_1>`, `<EMAIL_1>`) | Real values |
| Extra flags | `--pii-skip`, `--pii-only` | Not available |
| Confluence support | Yes | Yes |
| Config location | `~/.acli-pii/config.yaml` | OS keyring |
| Proxy site | `dt-rnd.atlassian.langdock.internal.dynatrace.com` | `dt-rnd.atlassian.net` |
