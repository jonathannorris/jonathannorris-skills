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

**4c. Draft the research block** — the findings only, in a neutral analytical voice. State what the code, spec, or history actually shows, with file paths, line numbers, and versions where they matter. Keep it tight and skip filler.

Two hard rules for this block:

- **Do not write it in the user's first person.** It is explicitly attributed to Claude, so phrasing like "I dug into this" or "I lean toward removing it" misrepresents who is speaking.
- **Do not make commitments on the user's behalf.** "I'll fix this", "happy to change it", and "your call" are the user's to say, not yours. Findings go in the research block; decisions and offers go in the user's own comment.

Do **not** apply `~/.config/GITHUB_VOICE_PROFILE.md` to the research block. That profile describes how the user writes, and this block is not the user writing. It applies only if the user explicitly asks you to draft their own comment for them.

**4d. Ask the user for their comment, then wait for approval** — every comment posted through this skill is a human comment written by the user, followed by the attributed research. Present the drafted research block, then ask what they want to say above it. Do NOT post until they approve. Watch for: `"post"` / `"ok"` (post as-is), inline corrections (apply then post), `"skip"` / `"next"` (move on).

If the user declines to add a comment, post the research block on its own with the attribution header intact. Never post research as though it were the user's own words.

**4e. Compose and post** — prefer inline replies using `in_reply_to`. Assemble the body in this format:

```markdown
<the user's comment, verbatim>

---

> [!NOTE]
> **🤖 Claude Code research**
>
> <research block>
```

Alert details: the research goes in a GitHub `> [!NOTE]` alert, which renders as a bordered callout while still rendering markdown inside, so keep inline backticks, bold, and links. Prefix every line with `> `, including blank lines as a bare `>`, or the alert terminates early. The `[!NOTE]` label and the attribution line are separate lines; do not merge them. The alert title is fixed and cannot be customised, which is why the attribution header sits inside the block.

Do not use a plain fenced block for the research. Nothing renders inside a fence and GitHub adds a horizontal scrollbar instead of wrapping.

If the research contains a table or its own fenced code block, leave those elements outside the alert; the attribution header still scopes them.

Write the composed body to a file and post it with `-F body=@<file>` rather than passing it inline. Bodies contain backticks and quotes, which get mangled when passed as a shell argument.

```bash
# Compose the body first (quoted heredoc, so nothing is interpolated)
cat > /tmp/pr-reply.md <<'BODY'
<composed body from the format above>
BODY

# Reply in an existing thread (most common)
gh api repos/{owner}/{repo}/pulls/{number}/comments \
  --method POST \
  -F body=@/tmp/pr-reply.md \
  -F in_reply_to=<parent_comment_id>

# New inline comment on a specific line
HEAD_SHA=$(gh pr view <number> --json headRefOid --jq '.headRefOid')
gh api repos/{owner}/{repo}/pulls/{number}/comments \
  --method POST \
  -F body=@/tmp/pr-reply.md \
  -f commit_id="$HEAD_SHA" \
  -f path='<file path>' \
  -F line=<line number> \
  -f side='RIGHT'

# General PR comment (last resort — no file/line context)
gh api repos/{owner}/{repo}/issues/{number}/comments \
  --method POST \
  -F body=@/tmp/pr-reply.md
```

Use `side='LEFT'` for deleted/original lines; `side='RIGHT'` for added or unchanged lines.

This format applies to every repo, including the OpenFeature org. There is no per-org exception.

**4f. Code changes** — make edits locally if the comment requires them. Do NOT commit mid-loop; batch all changes for Step 5.

For committing, pushing, CI checks, and incremental re-review, see [REFERENCE.md](REFERENCE.md).
