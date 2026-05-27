## Template Variables

In YAML/DQL files, use Go template syntax:

```yaml
# workflow.yaml
title: "{{.environment}} Deployment"
owner: "{{.team}}"
trigger:
  schedule:
    cron: "{{.schedule | default "0 0 * * *"}}"
```

```dql
# query.dql
fetch logs
| filter host.name == "{{.host}}"
| filter timestamp > now() - {{.timerange | default "1h"}}
```

Execute with: `dtctl apply -f file.yaml --set environment=prod --set team=platform`

## Copilot, Functions, Analyzers

```bash
dtctl get copilot-skills -o json --plain
dtctl get functions -o json --plain
dtctl exec function <id-or-name> --payload '{"key":"value"}' --plain
dtctl get analyzers -o json --plain
dtctl exec analyzer <id-or-name> --input '{"timeframe":"now-2h"}' --plain
```

## Authentication & Permissions

```bash
dtctl auth whoami --plain
dtctl auth can-i create workflows
dtctl auth can-i delete dashboards
```

## Common Issues

**Name resolution ambiguity:**
- If a name matches multiple resources, dtctl will fail
- Use IDs: `dtctl get <resource> -o json --plain | jq -r '.[] | "\(.id) | \(.name)"'`

**Permission denied:**
- Check token scopes: https://github.com/dynatrace-oss/dtctl/blob/main/docs/TOKEN_SCOPES.md
- Verify: `dtctl auth can-i <verb> <resource>`
- Check safety level: `dtctl config describe-context $(dtctl config current-context) --plain`

**Context/safety blocks:**
- Destructive operations may be blocked by safety level
- Switch context: `dtctl config use-context <name>`

## Safety Reminders

- Use `--plain` for machine/AI consumption
- Confirm context and safety level before destructive ops; prefer `get/describe` first
- Use `--mine` flag to filter resources you own
- For multi-tenant work, see [references/config-management.md](references/config-management.md)
