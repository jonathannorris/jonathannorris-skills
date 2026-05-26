---
name: bitbucket
description: Create, view, and manage pull requests on Bitbucket Server (Data Center) via the REST API. Use when asked to open a PR, list PRs, add reviewers, merge, comment, or interact with the Dynatrace internal Bitbucket instance at bitbucket.lab.dynatrace.org.
allowed-tools: Bash(curl:*) Bash(source:*)
---

# Bitbucket

## Quick start

```bash
source ~/.zshrc  # loads $BITBUCKET_TOKEN

# Create a PR
curl -s -X POST \
  "https://bitbucket.lab.dynatrace.org/rest/api/1.0/projects/PFS/repos/feature-management/pull-requests" \
  -H "Authorization: Bearer $BITBUCKET_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "feat: ICP-XXXX title",
    "description": "## Summary\n\n- bullet",
    "fromRef": {"id": "refs/heads/my-branch", "repository": {"slug": "feature-management", "project": {"key": "PFS"}}},
    "toRef":   {"id": "refs/heads/main",      "repository": {"slug": "feature-management", "project": {"key": "PFS"}}}
  }' | jq '{id, url: .links.self[0].href}'
```

## Workflows

### Create a PR
1. Confirm `fromRef` (source branch) and `toRef` (target branch)
2. Draft title and description — title must include a Jira key (e.g. `ICP-XXXX`) or the push hook rejects it
3. POST to `/pull-requests` (see quick start above)
4. Return the PR URL from `links.self[0].href`

### View / list PRs
```bash
# Single PR
curl -s "https://bitbucket.lab.dynatrace.org/rest/api/1.0/projects/{PROJECT}/repos/{REPO}/pull-requests/{ID}" \
  -H "Authorization: Bearer $BITBUCKET_TOKEN" | jq '{id, title, state}'

# All open PRs
curl -s "https://bitbucket.lab.dynatrace.org/rest/api/1.0/projects/{PROJECT}/repos/{REPO}/pull-requests?state=OPEN" \
  -H "Authorization: Bearer $BITBUCKET_TOKEN" | jq '.values[] | {id, title}'
```

### Comment on a PR
```bash
curl -s -X POST \
  "https://bitbucket.lab.dynatrace.org/rest/api/1.0/projects/{PROJECT}/repos/{REPO}/pull-requests/{ID}/comments" \
  -H "Authorization: Bearer $BITBUCKET_TOKEN" -H "Content-Type: application/json" \
  -d '{"text": "comment text"}' | jq '{id, text}'
```

### Merge a PR
Fetch the current `version` first — mutating operations require it:
```bash
VERSION=$(curl -s ".../pull-requests/{ID}" -H "Authorization: Bearer $BITBUCKET_TOKEN" | jq '.version')
curl -s -X POST ".../pull-requests/{ID}/merge?version=$VERSION" -H "Authorization: Bearer $BITBUCKET_TOKEN" | jq '{id, state}'
```

## Reference

See [REFERENCE.md](REFERENCE.md) for: add/remove reviewers, decline a PR, repo slug lookup, pagination.

## Known repos

| Repo | Project key | Slug |
|------|-------------|------|
| fm-service | PFS | feature-management |

To find others: `git -C <repo> remote get-url origin` — project key and slug are in the SSH URL.
