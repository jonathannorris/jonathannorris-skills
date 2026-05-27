---
name: dtctl
description: Investigate incidents, debug performance issues, analyze logs, and manage observability resources in Dynatrace using the dtctl CLI. Use this skill whenever the user asks about error rates, latency spikes, service health, crash-looping pods, web vitals, SLO status, open problems, root cause analysis, log patterns, trace analysis, or building dashboards — even if they don't mention Dynatrace by name. Also covers DQL queries, workflow management, notebook and dashboard creation, settings configuration, and any operations against a Dynatrace environment.
---

# Dynatrace Control with dtctl

## Recommended Initialization

```bash
dtctl commands --brief -o json        # discover all commands and flags
dtctl config current-context          # show current context
dtctl config describe-context $(dtctl config current-context) --plain  # safety level, URL
dtctl auth whoami --plain             # authenticated user
```

## DQL Reference Usage

Before writing or executing any DQL, consult `references/DQL-reference.md` and follow its documented syntax and templates. Prefer the reference over memory.

## Resources & Commands

### Available Resources

| Resource | Aliases |
|----------|---------|
| analyzer | analyzers |
| app | apps |
| bucket | bkt |
| copilot-skill | copilot-skills |
| dashboard | dash |
| edgeconnect | ec |
| extension | ext, extensions |
| extension-config | extcfg, extension-configs |
| function | fn, func |
| group | groups |
| intent | intents |
| lookup | lookups, lkup |
| notebook | nb |
| notification | notifications |
| sdk-version | sdk-versions |
| settings | setting |
| settings-schema | schema |
| slo | - |
| slo-template | slo-templates |
| trash | deleted |
| user | users |
| workflow | wf |
| workflow-execution | wfe |

Use IDs whenever possible instead of names to avoid ambiguity.

### Command Verbs

| Verb | Description | Example |
|------|-------------|---------|
| **get** | List resources | `dtctl get workflows --mine` |
| **describe** | Show resource details | `dtctl describe workflow <id>` |
| **edit** | Edit resource interactively | `dtctl edit dashboard <id>` |
| **apply** | Create/update from file | `dtctl apply -f workflow.yaml --set env=prod` |
| **delete** | Delete resource | `dtctl delete workflow <id>` |
| **exec** | Execute workflow/function/analyzer/copilot | `dtctl exec workflow <id>` |
| **query** | Run DQL query | `dtctl query "fetch logs \| limit 10"` |
| **logs** | Print resource logs | `dtctl logs workflow-execution <id>` |
| **wait** | Wait for conditions | `dtctl wait query "fetch logs" --for=any` |
| **history** | Show document history | `dtctl history dashboard <id>` |
| **restore** | Restore document version | `dtctl restore dashboard <id> --version 3` |
| **share** | Share document | `dtctl share dashboard <id> --user email@example.com` |
| **find** | Discover resources | `dtctl find intents --data trace.id=abc` |
| **open** | Open in browser | `dtctl open intent <app/intent> --data key=value` |
| **diff** | Compare resources | `dtctl diff -f workflow.yaml` |
| **verify** | Validate without executing | `dtctl verify query 'fetch logs' --fail-on-warn` |
| **commands** | List all commands | `dtctl commands --brief -o json` |

## Output Modes

For AI agents, prefer `--agent` (auto-detected) or `-o json --plain`:

```bash
dtctl <command> --agent        # structured JSON envelope: ok/result/error/context
dtctl <command> -o json --plain
dtctl <command> -o chart --plain   # ASCII chart for time series
```

## Quick Reference: DQL Queries

```bash
dtctl query "fetch logs | filter status='ERROR' | limit 100" -o json --plain
dtctl query -f query.dql --set host=h-123 --set timerange=2h -o json --plain
dtctl wait query "fetch spans | filter test_id='test-123'" --for=count=1 --timeout 5m
dtctl query "timeseries avg(dt.host.cpu.usage)" -o chart --plain
```

## Dashboards

Create/update: `dtctl apply -f dashboard.yaml --plain`. Export for reference: `dtctl get dashboard <id> -o yaml --plain`.

For the full YAML skeleton, tile types, visualizationSettings, and gotchas, see [references/resources/dashboards.md](references/resources/dashboards.md).

## Additional Resources

- **Troubleshooting / install**: [references/troubleshooting.md](references/troubleshooting.md)
- **Multi-tenant / context management**: [references/config-management.md](references/config-management.md)
- **DQL syntax and templates**: [references/DQL-reference.md](references/DQL-reference.md)
- **Notebooks**: [references/resources/notebooks.md](references/resources/notebooks.md)
- **Extensions**: [references/resources/extensions.md](references/resources/extensions.md)
- **Common issues and safety reminders**: [REFERENCE.md](REFERENCE.md)
