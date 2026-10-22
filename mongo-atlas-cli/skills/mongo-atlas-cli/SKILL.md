---
name: mongo-atlas-cli
description: Investigate MongoDB Atlas cluster and query performance with the read-only Atlas CLI (atlas). Covers listing projects, clusters and processes, pulling host metrics, Performance Advisor suggested indexes and namespaces, slow query logs, and alerts. Use when asked about MongoDB or Atlas performance, slow queries, missing indexes, query targeting, connection counts, or Atlas alerts, e.g. "why is mongo slow", "check the Atlas performance advisor", "what indexes does Atlas suggest", "show mongo connection metrics".
allowed-tools: Bash(atlas orgs list:*) Bash(atlas projects list:*) Bash(atlas clusters list:*) Bash(atlas clusters describe:*) Bash(atlas processes list:*) Bash(atlas metrics:*) Bash(atlas performanceAdvisor:*) Bash(atlas alerts list:*) Bash(atlas alerts describe:*)
---

## When to use this skill

- "Why is the mongo cluster slow?"
- "What indexes does the Atlas Performance Advisor suggest?"
- "Check query targeting / connections / opcounters on the primary"
- "Are there any open Atlas alerts?"

This skill is **read-only**. Never run `atlas` commands that create, update, pause, or delete anything (`create`, `update`, `delete`, `pause`, `start`, `slowOperationThreshold enable/disable`, `auth login`).

## Step 1: Check auth

The CLI reads a service account from `MONGODB_ATLAS_CLIENT_ID` and `MONGODB_ATLAS_CLIENT_SECRET`. If only `MONGO_CLIENT_ID` and `MONGO_CLIENT_SECRET` are set, map them first. Never echo the values.

```bash
[ -z "$MONGODB_ATLAS_CLIENT_ID" ] && export MONGODB_ATLAS_CLIENT_ID="$MONGO_CLIENT_ID" MONGODB_ATLAS_CLIENT_SECRET="$MONGO_CLIENT_SECRET"
atlas projects list -o json | jq -r '.results[] | "\(.id) \(.name) clusters=\(.clusterCount)"'
```

If the vars were just added to `~/.zshrc`, the agent's shell may not have them yet; run commands through `zsh -ic '...'` to pick them up.

## Step 2: Find the hosts

Most performance commands take a `hostname:port` process ID, usually the primary.

```bash
P=<projectId>
atlas clusters list --projectId $P -o json | jq -r '.results[] | "\(.name) \(.mongoDBVersion) \(.stateName)"'
atlas processes list --projectId $P -o json | jq -r '.results[] | "\(.id) \(.typeName)"'
H=$(atlas processes list --projectId $P -o json | jq -r '.results[] | select(.typeName=="REPLICA_PRIMARY") | .id' | head -1)
```

## Step 3: Metrics

`--granularity` accepts `PT10S`, `PT1M`, `PT5M`, `PT1H`, `P1D`. Use `--period` (e.g. `PT1H`, `P1D`) or `--start`/`--end`, not both.

```bash
atlas metrics processes $H --projectId $P --granularity PT5M --period PT6H \
  --type CONNECTIONS,OPCOUNTER_QUERY,QUERY_TARGETING_SCANNED_OBJECTS_PER_RETURNED,QUERY_EXECUTOR_SCANNED,PROCESS_CPU_USER,SYSTEM_NORMALIZED_CPU_USER \
  -o json | jq -r '.measurements[] | [.name, .units, ([.dataPoints[].value | select(.!=null)] | (max, .[-1]))] | @tsv'
```

Output columns: metric, units, max over the window, latest. Useful signals:

- `QUERY_TARGETING_SCANNED_OBJECTS_PER_RETURNED`: documents scanned per document returned. Close to 1 is good; Atlas alerts by default above 1000.
- `CONNECTIONS`: compare against the tier's connection limit.
- `atlas metrics databases list $H` and `atlas metrics disks list $H` list per-database and per-disk metrics.

## Step 4: Performance Advisor

```bash
atlas performanceAdvisor suggestedIndexes list --processName $H --projectId $P -o json \
  | jq '.suggestedIndexes[] | {namespace, index, weight, impact}'
atlas performanceAdvisor namespaces list --processName $H --projectId $P -o json | jq -r '.namespaces[] | "\(.namespace) \(.type)"'
```

Treat suggested indexes as leads, not instructions: check them against existing indexes and write volume before recommending one.

## Step 5: Slow query logs (needs extra role)

```bash
atlas performanceAdvisor slowQueryLogs list --processName $H --projectId $P --duration 3600000 --nLog 50 -o json
```

`--since` and `--duration` are in **milliseconds**. This command needs **Project Data Access Read/Write**; a Project Read Only or Observability Viewer account gets `Error: unauthorized`. If that happens, say so and fall back to suggested indexes and metrics rather than asking for a broader role.

Slow query log lines contain literal query values, which can be customer data. Summarize query shapes (namespace, filter keys, plan summary, duration) and do not paste raw values into responses.

## Step 6: Alerts

```bash
atlas alerts list --projectId $P --status OPEN -o json | jq -r '.results[] | "\(.created) \(.severity) \(.eventTypeName) \(.status) \(.hostnameAndPort // "")"'
```

## Permissions reference

| Command | Minimum Atlas project role |
|---|---|
| projects, clusters, processes, metrics, alerts | Project Read Only |
| suggestedIndexes, namespaces | Project Read Only |
| slowQueryLogs | Project Data Access Read/Write |
