---
name: pr-review
description: Fetch, analyze, and draft responses to GitHub PR review comments. Covers fetching comments via gh CLI, filtering bots, processing comments one-by-one with user approval before posting, and syncing code changes back to the branch.
---

## When to use this skill

Use this skill when asked to review, respond to, or process PR comments. This includes requests like:
- "help me review the comments on this PR"
- "review the PR comments on #63"
- "can you check for new comments on the PR"
- "help me respond to the review feedback"
- "process the latest PR review comments"

## Step 1: Fetch PR context

### Derive owner/repo

Always extract `owner/repo` from `git remote -v`, never from the local folder path:

```bash
git remote -v
# Example: origin  https://github.com/open-feature/protocol.git (fetch)
# Use "open-feature/protocol"
```

### Fetch PR metadata and diff

```bash
# PR metadata
gh pr view <number> --json title,body,baseRefName,headRefName,author,state

# Full diff for understanding the changes
gh pr diff <number>
```

If the user provides a full GitHub URL instead of a number, extract the PR number from it or pass the URL directly to `gh pr view` / `gh pr diff`.

### Fetch inline review comments

```bash
# All inline comments (paginated)
gh api repos/{owner}/{repo}/pulls/{number}/comments --paginate
```

### Fetch top-level reviews

```bash
# All reviews (paginated)
gh api repos/{owner}/{repo}/pulls/{number}/reviews --paginate
```

### Fetch review threads (GraphQL)

For richer thread context including resolution status:

```bash
gh api graphql -f query='
{
  repository(owner: "{owner}", name: "{repo}") {
    pullRequest(number: {number}) {
      reviewThreads(first: 100) {
        nodes {
          id
          isResolved
          isOutdated
          comments(first: 20) {
            nodes {
              id
              author { login }
              body
              path
              line
              createdAt
            }
          }
        }
      }
    }
  }
}'
```

### Filter for new comments only

If reviewing incrementally, filter by date:

```bash
gh api repos/{owner}/{repo}/pulls/{number}/comments --paginate \
  --jq '.[] | select(.created_at > "YYYY-MM-DDTHH:MM:SSZ") | {id, author: .user.login, path, line, created_at, body}'
```

## Step 2: Organize and triage comments

### Parse the fetched data

For each comment, extract:
- **Author** (`user.login`)
- **File and line** (`path`, `line`, `original_line`)
- **Body** (full text, no truncation)
- **Thread context** (`in_reply_to_id` for threaded replies)
- **Timestamp** (`created_at`)

### Group into threads

Group replies under their parent comment using `in_reply_to_id` or the GraphQL `reviewThreads` structure. Present threads together so the full conversation context is visible.

### Classify comments

Separate comments into:
- **Bot comments** -- `gemini-code-assist[bot]`, `cursor[bot]`, `copilot-review[bot]`, or any `user.type == "Bot"`
- **Human comments** -- everything else

Bot comments should still be reviewed and taken seriously — they often catch real issues. The distinction matters mainly for how you respond: bot threads usually don't need a posted reply once addressed or dismissed.

### Filter out already-resolved threads

If using GraphQL, skip threads where `isResolved == true` unless the user specifically asks to review all threads.

## Step 3: Handle bot comments

Review each bot comment on its merits — bots frequently surface real issues worth addressing. The difference from human comments is in the response: once you've reviewed or addressed a bot thread, resolve it rather than posting a reply. Posting "thanks" or "fixed" comments back to bots is usually noise.

Present bot comments to the user the same way you would human ones (see Step 4a) when they raise substantive concerns, so the user can decide whether to address them in code. For bot comments that are clearly non-actionable (style nits already handled, false positives, etc.), summarize them briefly and resolve.

Resolve bot review threads using the GraphQL mutation:

```bash
gh api graphql -f query='
mutation {
  resolveReviewThread(input: {threadId: "<thread_id>"}) {
    thread {
      isResolved
    }
  }
}'
```

Default to resolving bot threads without posting a reply. Only post a reply to a bot thread if the user explicitly asks for one.

## Step 4: Process human comments one-by-one

This is the core loop. Work through each unresolved human comment thread sequentially:

### 4a. Present the comment

Show the user:
- Who wrote it and when
- Which file and line it refers to
- The full comment body (and any prior replies in the thread)
- The relevant code context from the diff

### 4b. Research if needed

Before drafting a response, investigate the commenter's concern:
- Read the relevant section of the changed file
- Check related specs, ADRs, or documentation in the repo
- Look at related implementations in sibling repos if the user has pointed them out
- If the comment references an external standard or pattern, look it up

### 4c. Draft a response

Draft a **short, direct** response. Guidelines:
- Keep responses concise; one to three sentences is usually enough
- Be direct and technical, not performative
- If the commenter is right, acknowledge it briefly and describe the fix
- If you disagree, explain why with specific reasoning
- If a code or doc change is needed, note what you plan to change

**Voice profile:** Before drafting your first response, read `~/.config/GITHUB_VOICE_PROFILE.md` if it exists. Follow the tone, phrasing patterns, and style described there for all GitHub-facing content.

### 4d. Wait for user approval

Present the drafted response and wait. Do NOT post until the user explicitly approves. Watch for:
- `"post"` / `"post it"` / `"yea"` / `"ok"` -- post the response as-is
- Inline corrections -- the user may adjust wording, tone, or add context before approving
- `"skip"` / `"next"` -- move to the next comment without posting

### 4e. Post the response

Reply in the existing review thread:

```bash
gh api repos/{owner}/{repo}/pulls/{number}/comments \
  --method POST \
  -f body='<response text>' \
  -F in_reply_to=<parent_comment_id>
```

For top-level PR comments (not inline):

```bash
gh api repos/{owner}/{repo}/issues/{number}/comments \
  --method POST \
  -f body='<response text>'
```

### 4f. Make code/doc changes if needed

If the comment requires a change to the PR's files:
1. Make the edit locally
2. Tell the user what was changed
3. Continue to the next comment

Do NOT commit mid-loop. Batch all file changes and commit once at the end (Step 5).

### 4g. Move to the next comment

After posting (or skipping), present the next unresolved comment and repeat from 4a.

## Step 5: Sync changes back

After all comments have been processed:

### Summarize changes made

List all file edits that resulted from the review feedback.

### Commit and push (with approval)

IMPORTANT: Do NOT commit or push unless the user explicitly requests it.

```bash
git add <changed files>
git commit -m "docs: address PR review feedback" -s   # -s if repo requires sign-off
git push
```

Use conventional commit format. If multiple distinct changes were made, a single commit summarizing all review feedback is fine unless the user prefers separate commits.

### Verify CI

After pushing:

```bash
gh pr checks <number>
```

If checks fail, investigate and offer to fix.

## Step 6: Incremental re-review

The user may come back later to check for new comments. When asked to re-review:

1. Fetch comments filtered by date (after the last review timestamp)
2. Skip already-resolved threads
3. Repeat the one-by-one processing loop from Step 4

## Important reminders

- Always derive `owner/repo` from `git remote -v`, not from the folder path
- Never post a comment without explicit user approval
- Keep responses short; the user's style is direct and concise
- Review bot comments on their merits; they often catch real issues. Resolve bot threads instead of posting replies, unless the user asks otherwise
- Read the voice profile (`~/.config/GITHUB_VOICE_PROFILE.md`) before drafting any responses
- If a commenter's concern is unclear, research it before drafting a response rather than guessing
- When the user provides business context or personal experience to include in a response, weave it in naturally
- If the user pastes screenshots of comments, extract the content and process them like any other comment
