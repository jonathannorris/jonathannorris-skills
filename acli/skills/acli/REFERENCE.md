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

## Atlassian Document Format (ADF)

Jira stores rich text as ADF, not Markdown. To get real headings, bullets, and inline code into a description or comment, build an ADF document and pass it with `--description-file` (or `--body-file` for comments). See the "Formatting descriptions and comments" section in [SKILL.md](SKILL.md) for when to reach for this.

### Document shape

Every document is `{"type": "doc", "version": 1, "content": [...]}`. The nodes worth knowing:

| Node | JSON |
|---|---|
| Paragraph | `{"type":"paragraph","content":[{"type":"text","text":"..."}]}` |
| Heading | `{"type":"heading","attrs":{"level":3},"content":[...]}` |
| Bullet list | `{"type":"bulletList","content":[ <listItem>, ... ]}` |
| Numbered list | `{"type":"orderedList","content":[ <listItem>, ... ]}` |
| List item | `{"type":"listItem","content":[ <paragraph> ]}` |
| Code block | `{"type":"codeBlock","attrs":{"language":"go"},"content":[{"type":"text","text":"..."}]}` |
| Rule | `{"type":"rule"}` |

Marks decorate a text node via a `marks` array: `code`, `strong`, `em`, `strike`, and
`{"type":"link","attrs":{"href":"https://..."}}`.

Two constraints that are easy to trip over:

- A `listItem` must wrap its text in a `paragraph`; text cannot sit directly in the item.
- Inline code is a mark on a text node, so mixed text splits into several nodes. `Set the flag now` with `flag` in code is three text nodes, not one.

### Template

```json
{
  "type": "doc",
  "version": 1,
  "content": [
    {"type": "paragraph", "content": [{"type": "text", "text": "Opening summary sentence."}]},
    {"type": "heading", "attrs": {"level": 3}, "content": [{"type": "text", "text": "Scope"}]},
    {"type": "bulletList", "content": [
      {"type": "listItem", "content": [
        {"type": "paragraph", "content": [
          {"type": "text", "text": "Set "},
          {"type": "text", "text": "timeout", "marks": [{"type": "code"}]},
          {"type": "text", "text": " to 4s."}
        ]}
      ]},
      {"type": "listItem", "content": [
        {"type": "paragraph", "content": [{"type": "text", "text": "Second item."}]}
      ]}
    ]},
    {"type": "paragraph", "content": [
      {"type": "text", "text": "See the ", "marks": []},
      {"type": "text", "text": "design doc", "marks": [{"type": "link", "attrs": {"href": "https://example.com"}}]},
      {"type": "text", "text": "."}
    ]}
  ]
}
```

### Editing an existing description

Keep the ADF file next to the work you are doing and edit the JSON rather than retyping the
description. Removing a section means deleting its heading node and the node that follows it:

```bash
python3 - <<'EOF'
import json
doc = json.load(open('desc.json'))
c = doc['content']
i = next(i for i, n in enumerate(c) if n['type'] == 'heading' and n['content'][0]['text'] == 'Out of scope')
del c[i:i+2]          # heading plus its list
json.dump(doc, open('desc.json', 'w'), indent=2)
EOF
acli-pii jira workitem edit --key ICP-123 --description-file desc.json --yes
```

### Bulk edits

`--generate-json` writes a skeleton work item definition, and `--from-json` applies it. Use these
when creating or editing several items in one pass; the `description` field inside that JSON takes
the same ADF document shown above.

```bash
acli-pii jira workitem create --generate-json
acli-pii jira workitem create --from-json workitem.json
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
