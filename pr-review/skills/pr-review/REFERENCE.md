## Step 5: Sync changes back

After all comments are processed, summarize file edits made during the review, then commit and push only when the user explicitly requests it:

```bash
git add <changed files>
git commit -m "docs: address PR review feedback" -s   # -s if repo requires sign-off
git push
```

Use conventional commit format. A single commit summarizing all review feedback is fine unless the user prefers separate commits.

After pushing, verify CI:

```bash
gh pr checks <number>
```

If checks fail, investigate and offer to fix.

## Step 6: Incremental re-review

When asked to check for new comments:

1. Fetch comments filtered by date (after the last review timestamp):
   ```bash
   gh api repos/{owner}/{repo}/pulls/{number}/comments --paginate \
     --jq '.[] | select(.created_at > "YYYY-MM-DDTHH:MM:SSZ") | {id, author: .user.login, path, line, created_at, body}'
   ```
2. Skip already-resolved threads
3. Repeat the one-by-one processing loop from Step 4
