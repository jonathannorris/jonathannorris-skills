---
name: openfeature-community-updates-blog
description: Draft, fact-check, and publish an OpenFeature community update blog post on openfeature.dev, covering source gathering across the org, vendor-neutral voice, batched GitHub verification of every PR/issue/release claim, link checking, and the worktree-to-draft-PR publish flow. Use when asked to write or update an OpenFeature blog post, community update, mid-year or quarterly recap, ecosystem roundup, or release announcement -- e.g. "draft the mid-2026 community update", "write a blog post about what shipped this quarter", "add the new vendors to the update post", "turn these community notes into a blog post".
---

# OpenFeature community update blog post

Repo: `~/git/OpenFeature/openfeature.dev`. Posts live in `blog/YYYY-MM-DD-<slug>.md`.

## Step 1: Set up an isolated worktree

Never draft in the main checkout. See [REFERENCE.md](REFERENCE.md) for the worktree commands and the `.worktrees/` convention.

## Step 2: Gather source material

Pull from primary sources, not memory:

- `~/git/OpenFeature/community` and the `openfeature-community-updates` repo for meeting notes and periodic updates
- GitHub releases, merged PRs, and open umbrella issues across the org (`spec`, `flagd`, `protocol`, each `*-sdk`, each `*-sdk-contrib`, `open-feature-operator`)
- `src/datasets/providers/*.ts` in openfeature.dev: **authoritative** for vendor names, per-language providers, official-vs-community status, and links
- Open PRs on openfeature.dev itself (`gh pr list`) for providers that shipped but are not listed yet
- Vendor blogs and docs for platform launches, fetched and quoted, never paraphrased from memory

Inherited notes may carry errors (issues cited as merged PRs, renamed repos). Verify before repeating.

## Step 3: Study house style before writing

Read the two or three most recent posts in `blog/`. Non-negotiables:

- **One sentence per line.** Hard requirement, matches `max-one-sentence-per-line` lint.
- **No em dashes** anywhere. Use commas, colons, semicolons, or `--` in headings.
- Frontmatter: `title`, `description`, `date`, `categories`, `tags`, `slug`, `authors`, optional `image`
- `<!--truncate-->` after the two-line intro
- Internal links as routes (`/blog/<slug>`, `/docs/...`, `/community/...`), never full URLs

## Step 4: Write in a vendor-neutral voice

OpenFeature is a CNCF project, so the post must not read as marketing for anyone. See [REFERENCE.md](REFERENCE.md) for the full rules and the proven section structure. The short version:

- No rankings, superlatives, or primacy claims ("first", "broadest", "most complete")
- Attribute vendor claims to the vendor's own wording, in quotes
- State the ordering you chose (chronological, alphabetical) so it reads as neutral
- Distinguish vendor-maintained providers from community providers in the OpenFeature org
- Keep bullets to a sentence or two; verifying something deeply is not a reason to write it all up

## Step 5: Verify every claim

Every PR, issue, release, version number, and count gets checked against GitHub before it ships. Batch it: one `gh api graphql` call can confirm twenty PR states. See [REFERENCE.md](REFERENCE.md) for the query patterns and the link-checking recipe.

Report factual problems in the author's own edits rather than silently fixing them.

## Step 6: Publish

Signed-off conventional commit, push, open a **draft** PR. The Netlify deploy preview is the real internal-link check, because `docusaurus.config.ts` sets `onBrokenLinks: 'throw'`. Blog posts are **not** covered by CI markdownlint (its glob says `blogs/**`, a typo), so lint locally with care. Full checklist in [REFERENCE.md](REFERENCE.md).

## Step 7: Iterate on feedback

Feedback arrives from Slack screenshots, PR review comments, and the author's own commits. Re-verify against primary sources before applying: reviewer suggestions are often directionally right but numerically off. Push each round as its own small commit.
