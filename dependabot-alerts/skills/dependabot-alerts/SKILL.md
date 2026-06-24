---
name: dependabot-alerts
description: Fix open Dependabot security alerts across any repo. Covers fetching alerts via gh CLI, fixing vulnerabilities per ecosystem (npm, pip, Go, Java, Swift, GitHub Actions), and creating a PR with structured alert resolution details. Use when asked to fix, resolve, or address Dependabot alerts — e.g. "fix all the open dependabot alerts", "resolve dependabot security alerts".
---

## Step 1: Setup

### Determine the correct GitHub owner/repo

CRITICAL: Always extract owner/repo from git remote, not the local folder name:

```bash
git remote -v
# Example: origin  https://github.com/DevCycleHQ/cli.git (fetch)
# Use "DevCycleHQ/cli"
```

### Check for existing PRs addressing these alerts

Before creating a branch or doing any work, check whether an open PR already exists for these alerts:

```bash
# Check for open PRs on the dependabot-alerts branch
gh pr list --repo {owner}/{repo} --state open --head chore/dependabot-alerts --json number,title,url
gh pr list --repo {owner}/{repo} --state open --head fix/dependabot-alerts --json number,title,url

# Also check Dependabot's own automated PRs
gh pr list --repo {owner}/{repo} --state open --author app/dependabot --json number,title,url --limit 20
```

If an open PR on `chore/dependabot-alerts` or `fix/dependabot-alerts` already exists (i.e. a PR we previously opened, not a Dependabot-generated PR):
- Switch to that branch and update it with any new alert fixes rather than opening a new PR
- Note the existing PR URL in your summary

Dependabot's own automated PRs (author `app/dependabot`) should be left alone — do not modify or close them, and do not duplicate their fixes in our PR. Mention them in our PR description as related work (e.g. "Note: #123 is a Dependabot PR covering `lodash`").

### Create a clean branch off the default branch

```bash
DEFAULT_BRANCH="$(gh repo view {owner}/{repo} --json defaultBranchRef --jq '.defaultBranchRef.name')"
git fetch origin "$DEFAULT_BRANCH"
git checkout -b chore/dependabot-alerts "origin/$DEFAULT_BRANCH"
```

If `chore/dependabot-alerts` already exists locally or on remote (and there is no open PR to update), use `chore/dependabot-alerts-2`.

## Step 2: Fetch open alerts

```bash
gh api --paginate --slurp repos/{owner}/{repo}/dependabot/alerts \
  --jq 'map(.[]) | map(select(.state == "open")) | .[] | {
    number: .number,
    package: .dependency.package.name,
    ecosystem: .dependency.package.ecosystem,
    severity: .security_advisory.severity,
    manifest: .dependency.manifest_path,
    vulnerable_range: .security_vulnerability.vulnerable_version_range,
    patched: .security_vulnerability.first_patched_version.identifier
  }'
```

If `patched` is null, check `.security_advisory.vulnerabilities[]` for `first_patched_version`.

## Step 3: Fix vulnerabilities by ecosystem

See [REFERENCE.md](REFERENCE.md) for ecosystem-specific instructions: npm/yarn/pnpm, pip, Go, Java, Swift, .NET, Ruby, GitHub Actions.

**Key rules for JS (most common):**
1. Try a lockfile-only bump first (`yarn upgrade <pkg>`, `npm update <pkg>`) — preferred, less intrusive
2. For dev/test transitive deps: bump the direct dev dep that ships the fix
3. For production transitive deps: ask the user before adding overrides/resolutions
4. Only add `resolutions`/`overrides` to `package.json` when the patched version is outside every existing semver range

## Step 4: Verify fixes

- Run the project's test suite if one exists
- Confirm manifest and lockfile resolve to the patched version for each alert
- Build the project to catch compilation errors

## Step 5: Commit

```
chore: resolve open dependabot security alerts
```

Do NOT use `#<number>` to reference alerts — GitHub resolves those to issues/PRs. Use full alert URLs if needed: `https://github.com/{owner}/{repo}/security/dependabot/<num>`.

Follow AGENTS.md — do NOT commit unless the user explicitly requests it.

## Step 6: Create the PR

Draft branch name, commit message, PR title, and PR body, then present to the user for review before pushing.

**PR title:** `chore: resolve open dependabot security alerts`

**PR body:** Short `## Summary` with 1-3 bullets: overall scope, any cross-cutting changes, any unresolvable alerts. No per-alert tables or exhaustive lists.

```bash
gh pr create --title "chore: resolve open dependabot security alerts" --body "..." --reviewer DevCycleHQ/engineering
```

## Step 7: Post-PR checks

After creating the PR, request the engineering team as reviewer if not already requested:

```bash
gh pr edit <pr_number> --add-reviewer DevCycleHQ/engineering
```

Then check CI:

```bash
gh pr checks <pr_number>
```

Common CI failures: ESLint major version bump, Python version constraints, TypeScript type changes, snapshot regeneration. Fix, commit, push (no force push).
