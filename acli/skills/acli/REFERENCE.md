## Setup

### Binary

`acli` is installed via Homebrew. Verify with:

```bash
acli version
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

### Building ADF from Python

Hand-written ADF gets verbose fast, and regexes or Windows paths need double escaping inside JSON. Build the document with small helpers instead; raw strings (`r"..."`) keep backslashes intact:

```python
import json, sys
def t(s): return {"type": "text", "text": s}
def c(s): return {"type": "text", "text": s, "marks": [{"type": "code"}]}
def p(*n): return {"type": "paragraph", "content": list(n)}
def li(*n): return {"type": "listItem", "content": [p(*n)]}
def ul(*items): return {"type": "bulletList", "content": list(items)}

doc = {"type": "doc", "version": 1, "content": [
    p(t("Checked "), c("app:schema-id"), t(". No change needed.")),
    ul(
        li(c("token"), t(": already type "), c("secret"), t(".")),
        li(c("host"), t(": restricted by "), c(r"^[a-z0-9]+\.example\.com$"), t(".")),
    ),
]}
json.dump(doc, open(sys.argv[1], "w"))
```

```bash
python3 build_comment.py comment.json && acli jira workitem comment create --key ICP-123 --body-file comment.json
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
acli jira workitem edit --key ICP-123 --description-file desc.json --yes
```

### Bulk edits

`--generate-json` writes a skeleton work item definition, and `--from-json` applies it. Use these
when creating or editing several items in one pass; the `description` field inside that JSON takes
the same ADF document shown above.

```bash
acli jira workitem create --generate-json
acli jira workitem create --from-json workitem.json
```

## Scripting with JSON + jq

```bash
# Extract a single field
acli jira workitem view ICP-123 --json | jq '.fields.status.name'

# Get all keys from a search
acli jira workitem search --jql "project = ICP AND status = 'To Do'" --json \
  | jq -r '.[].key'

# Export to CSV
acli jira workitem search --jql "project = ICP" --csv > results.csv
```
