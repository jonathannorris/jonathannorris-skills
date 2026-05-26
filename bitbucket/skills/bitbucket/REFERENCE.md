# Bitbucket REST API Reference

Base URL: `https://bitbucket.lab.dynatrace.org/rest/api/1.0`  
Auth: `Authorization: Bearer $BITBUCKET_TOKEN` (set in `~/.zshrc`)

## Add a reviewer

Fetch current version and reviewer list, then PUT with the new reviewer appended:

```bash
PR=$(curl -s ".../pull-requests/{ID}" -H "Authorization: Bearer $BITBUCKET_TOKEN")
VERSION=$(echo $PR | jq '.version')
REVIEWERS=$(echo $PR | jq '[.reviewers[].user | {name: .name}]')

curl -s -X PUT ".../pull-requests/{ID}" \
  -H "Authorization: Bearer $BITBUCKET_TOKEN" -H "Content-Type: application/json" \
  -d "{\"version\": $VERSION, \"reviewers\": $(echo $REVIEWERS | jq '. + [{\"user\": {\"name\": \"USERNAME\"}}]')}" \
  | jq '{id, reviewers: [.reviewers[].user.displayName]}'
```

## Decline a PR

```bash
VERSION=$(curl -s ".../pull-requests/{ID}" -H "Authorization: Bearer $BITBUCKET_TOKEN" | jq '.version')
curl -s -X POST ".../pull-requests/{ID}/decline?version=$VERSION" \
  -H "Authorization: Bearer $BITBUCKET_TOKEN" | jq '{id, state}'
```

## Pagination

Responses include `isLastPage` and `nextPageStart`. Add `?start={nextPageStart}` to fetch the next page.

```bash
curl -s ".../pull-requests?state=OPEN&start=25" -H "Authorization: Bearer $BITBUCKET_TOKEN"
```

## API conventions

- Project keys are uppercase (`PFS`)
- Repo slugs are lowercase-hyphenated (`feature-management`)
- Mutating operations (PUT, merge, decline) require the current `version` field — always fetch first
- PR commit messages must include a Jira key or the pre-receive hook rejects the push
