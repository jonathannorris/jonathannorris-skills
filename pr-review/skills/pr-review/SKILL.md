---
name: pr-review
description: Fetch, analyze, and draft responses to GitHub PR review comments. Covers fetching comments via gh CLI, filtering bots, processing comments one-by-one with user approval before posting, and syncing code changes back to the branch. Use when asked to review, respond to, or process PR comments — e.g. "help me review the comments on this PR", "review PR #63", "help me respond to the review feedback".
---

## Step 1: Fetch PR context

Always extract `owner/repo` from `git remote -v`, never from the local folder path.

```bash
# PR metadata and diff
gh pr view <number> --json title,body,baseRefName,headRefName,author,state
gh pr diff <number>

# Inline review comments
gh api repos/{owner}/{repo}/pulls/{number}/comments --paginate

# All reviews
gh api repos/{owner}/{repo}/pulls/{number}/reviews --paginate

# Review threads with resolution status (GraphQL)
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

If the user provides a full GitHub URL, extract the PR number or pass the URL directly to `gh pr view`.

## Step 2: Organize and triage comments

- Group replies under their parent using `in_reply_to_id` or GraphQL `reviewThreads`
- Classify: **Bot** (`user.type == "Bot"`) vs **Human**
- Skip `isResolved == true` threads unless the user asks to see all

## Step 3: Handle bot comments

Review bot comments on their merits — they often catch real issues. Once reviewed or addressed, resolve the thread rather than posting a reply:

```bash
gh api graphql -f query='
mutation {
  resolveReviewThread(input: {threadId: "<thread_id>"}) {
    thread { isResolved }
  }
}'
```

Never resolve human comment threads — only the user does that.

## Step 4: Process human comments one-by-one

For each unresolved human thread:

**4a. Present** — author, file/line, full comment body, relevant code context from the diff

**4b. Research** — read the relevant file section, check specs/ADRs, look up external references if needed

**4c. Draft a response** — short and direct, 1-3 sentences. Before drafting your first response, read `~/.config/GITHUB_VOICE_PROFILE.md` if it exists.

**4d. Wait for user approval** — do NOT post until the user explicitly approves. Watch for: `"post"` / `"ok"` (post as-is), inline corrections (apply then post), `"skip"` / `"next"` (move on).

**4e. Post** — prefer inline replies using `in_reply_to`.

Every comment posted through this skill is published by an AI agent workflow, so **prefix the body with `bot: `** (literal `bot:` followed by a space, then the response). This must be applied to all posted comments — inline replies, new inline comments, and general PR comments alike. The user's drafted/approved text goes after the prefix.

```bash
# Reply in an existing thread (most common)
gh api repos/{owner}/{repo}/pulls/{number}/comments \
  --method POST \
  -f body='bot: <response>' \
  -F in_reply_to=<parent_comment_id>

# New inline comment on a specific line
HEAD_SHA=$(gh pr view <number> --json headRefOid --jq '.headRefOid')
gh api repos/{owner}/{repo}/pulls/{number}/comments \
  --method POST \
  -f body='bot: <response>' \
  -f commit_id="$HEAD_SHA" \
  -f path='<file path>' \
  -F line=<line number> \
  -f side='RIGHT'

# General PR comment (last resort — no file/line context)
gh api repos/{owner}/{repo}/issues/{number}/comments \
  --method POST \
  -f body='bot: <response>'
```

Use `side='LEFT'` for deleted/original lines; `side='RIGHT'` for added or unchanged lines.

**4f. Code changes** — make edits locally if the comment requires them. Do NOT commit mid-loop; batch all changes for Step 5.

For committing, pushing, CI checks, and incremental re-review, see [REFERENCE.md](REFERENCE.md).
