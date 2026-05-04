---
name: dependabot-alerts
description: Fix open Dependabot security alerts across any repo. Covers fetching alerts via gh CLI, fixing vulnerabilities per ecosystem (npm, pip, Go, Java, Swift, GitHub Actions), and creating a PR with structured alert resolution details.
---

## When to use this skill

Use this skill when asked to fix, resolve, or address Dependabot alerts or security vulnerabilities in a repository. This includes requests like:
- "fix all the open dependabot alerts"
- "create a new branch from the default branch, fix all the open dependabot alerts"
- "resolve dependabot security alerts"

## Step 1: Setup

### Determine the correct GitHub owner/repo

CRITICAL: The local folder name often differs from the GitHub org. Always extract the owner/repo from git remote:

```bash
git remote -v
# Example output: origin  https://github.com/DevCycleHQ/cli.git (fetch)
# Use "DevCycleHQ/cli" — NOT the local folder path
```

### Determine the default branch, pull latest, and create branch

Always create a clean branch directly off `origin/<default>`. Do NOT base it on any existing local branch.

```bash
DEFAULT_BRANCH="$(gh repo view {owner}/{repo} --json defaultBranchRef --jq '.defaultBranchRef.name')"
git fetch origin "$DEFAULT_BRANCH"
git checkout -b fix/dependabot-alerts "origin/$DEFAULT_BRANCH"
```

Do not assume the default branch is `main`; it may be `master`, `trunk`, or something else.

If a `fix/dependabot-alerts` branch already exists, use `fix/dependabot-alerts-2` (or increment the suffix).

If you are already on an existing branch that has unrelated commits ahead of the default branch, do not use it. Always start fresh from `origin/<default>`.

## Step 2: Fetch open alerts

### List all open alerts

```bash
gh api --paginate --slurp repos/{owner}/{repo}/dependabot/alerts \
  --jq 'map(.[]) | map(select(.state == "open")) | .[] | {
    number: .number,
    package: .dependency.package.name,
    ecosystem: .dependency.package.ecosystem,
    severity: .security_advisory.severity,
    summary: .security_advisory.summary,
    manifest: .dependency.manifest_path,
    vulnerable_range: .security_vulnerability.vulnerable_version_range,
    patched: .security_vulnerability.first_patched_version.identifier
  }'
```

### Get detailed info for a specific alert

```bash
gh api repos/{owner}/{repo}/dependabot/alerts/{number} \
  --jq '{
    number: .number,
    severity: .security_advisory.severity,
    summary: .security_advisory.summary,
    package: .dependency.package.name,
    manifest: .dependency.manifest_path,
    cve: .security_advisory.cve_id,
    vulnerable_range: .security_vulnerability.vulnerable_version_range,
    patched: .security_vulnerability.first_patched_version.identifier
  }'
```

If the `patched` field is null, check the advisory's vulnerabilities array:

```bash
gh api repos/{owner}/{repo}/dependabot/alerts/{number} \
  --jq '.security_advisory.vulnerabilities[] | {
    vulnerable_range: .vulnerable_version_range,
    patched: .first_patched_version
  }'
```

### Count alerts

```bash
gh api --paginate --slurp repos/{owner}/{repo}/dependabot/alerts \
  --jq 'map(.[]) | map(select(.state == "open")) | length'
```

## Step 3: Fix vulnerabilities by ecosystem

### npm / yarn / pnpm (JavaScript/TypeScript)

1. **Trace the dependency tree** to determine if the vulnerable package is direct or transitive:
   ```bash
   yarn why <package>    # yarn
   npm ls <package>      # npm
   pnpm why <package>    # pnpm
   ```

2. **Direct dependency:** Bump the version in `package.json` `dependencies` or `devDependencies`.

3. **Transitive dependency:** The right approach depends on whether the vulnerable package is a dev/test dependency or a production dependency.

   **Dev/test transitive deps — bump the direct dev dep:**
   For transitive deps that only affect development (test runners, linters, build tools, etc.), bump the direct dev dependency that brings in the vulnerable package to a version that ships the patched transitive dep:
   ```bash
   npm view "<parent>@latest" dependencies   # check what transitive version it ships
   ```
   Update the version in `devDependencies` and regenerate the lockfile. This is the cleanest approach — explicit, readable, and survives full lockfile regenerations.

   **Production transitive deps — ask the user:**
   If the vulnerable package is a production dependency (shipped to consumers of the package), ask the user how they want to handle it before proceeding. Options include:
   - Bumping the direct parent to a version that ships the patched transitive dep
   - Adding a permanent override/resolution in `package.json`
   - Accepting the risk if the vulnerability is not exploitable in the current usage

   Check if a newer parent version already ships the fix:
   ```bash
   npm view "<parent>@latest" dependencies   # check what transitive version it ships
   ```

   **When overrides in `package.json` are needed** (e.g. yarn, pnpm, repos without a committed lockfile, or production deps after user approval — see lockfile-pinning trick below as an alternative for npm):
   - **yarn (v1/classic):** Add to `resolutions` in `package.json`:
     ```json
     "resolutions": {
       "<package>": "^<patched_version>"
     }
     ```
   - **npm:** Add to `overrides` in `package.json`:
     ```json
     "overrides": {
       "<package>": "^<patched_version>"
     }
     ```
   - **pnpm:** Add to `pnpm.overrides` in `package.json`:
     ```json
     "pnpm": {
       "overrides": {
         "<package>": "^<patched_version>"
       }
     }
     ```
   Match the version range style used by the dependent package — if it uses `^`, your resolution should too.

   **npm lockfile-pinning trick (alternative to permanent overrides):** If you want to avoid leaving overrides in `package.json` for npm projects with a committed lockfile:
   1. Add the override temporarily to `package.json`
   2. Run `npm install` — this pins the patched version into the lockfile
   3. Remove the override from `package.json`
   4. Run `npm install` again to sync
   5. Verify with `npm ls <package>` that the patched version is still resolved

   The lockfile retains the pinned version across `npm ci` installs. Note: the pin will drift if someone runs `npm update <package>` or fully regenerates the lockfile.

4. **Regenerate the lockfile:**
   ```bash
   yarn install    # yarn
   npm install     # npm
   pnpm install    # pnpm
   ```

5. **Verify resolution:** Confirm the manifest and lockfile now resolve to the patched version for the vulnerable package. Use the Dependabot alert details as the source of truth rather than `npm audit`/`yarn audit`/`pnpm audit`.

6. **Monorepos:** Check for multiple `package.json` files. Each workspace may need its own resolution entry. Use glob to find all `package.json` files and check which ones reference the vulnerable package.

### pip (Python)

1. **Identify the manifest** from the alert's `manifest` field (e.g., `requirements.test.txt`, `example/django-app/requirements.txt`).

2. **Bump the version pin** in the relevant requirements file:
   ```
   # Before: black~=25.1.0
   # After:  black~=26.3.1
   ```

3. **Check for Python version constraints.** If the patched version requires a newer Python than the CI matrix supports, consider:
   - Splitting deps into separate requirements files (e.g., `requirements.lint.txt`)
   - Adjusting CI workflow Python versions

4. **Check `pyproject.toml` and `setup.cfg`** for version constraints that may also need updating.

### Go

1. **Update the vulnerable module:**
   ```bash
   go get <module>@latest
   go mod tidy
   ```

2. **Verify:** Confirm `go.mod` and `go.sum` now resolve to the patched module version, then run `go build ./...` and `go test ./...`

### Java / Gradle / Maven

1. **Gradle:** Update dependency versions in `build.gradle` or `build.gradle.kts`. For transitive deps, use dependency constraints or force resolution:
   ```groovy
   configurations.all {
       resolutionStrategy {
           force 'group:artifact:version'
       }
   }
   ```

2. **Maven:** Update version in `pom.xml` or use `<dependencyManagement>` for transitive overrides.

3. **Verify:** `./gradlew build` or `mvn verify`

### Swift

1. Update pinned versions in `Package.resolved`.
2. Run `swift package resolve` and `swift build`.

### .NET / NuGet

1. Update package versions in `.csproj` files or `Directory.Packages.props`.
2. Run `dotnet restore` and `dotnet build`.

### Ruby / Bundler

1. Update version constraints in `Gemfile`.
2. Run `bundle update <gem>` and confirm `Gemfile.lock` now resolves to a patched version for the affected gem.

### GitHub Actions

1. Bump action versions in `.github/workflows/*.yml` files:
   ```yaml
   # Before: uses: actions/checkout@v4
   # After:  uses: actions/checkout@v5
   ```
2. Verify workflows still function by checking CI after push.

## Step 4: Verify fixes

- Run the project's test suite if one exists.
- Use the Dependabot alerts themselves as the source of truth for whether the vulnerability is resolved.
- Confirm the affected manifest and lockfile now resolve to a patched version for each alert you addressed.
- Build the project to catch compilation errors.

## Step 5: Commit

Use conventional commit format with the `chore` type, since dependency bumps are maintenance work:

```
chore: resolve open dependabot security alerts
```

For commits addressing multiple alerts, include a body listing the changes:

```
chore: resolve open dependabot security alerts

- <package> <old_version> -> <new_version> (<severity>, Dependabot alert <number>)
- <package> <old_version> -> <new_version> (<severity>, Dependabot alert <number>)
```

IMPORTANT: Do NOT use `#<number>` in commit messages or PR bodies when referring to Dependabot alerts — GitHub auto-links `#N` to issues/PRs, not security alerts. Use the full alert URL instead (see PR body format below).

IMPORTANT: Follow the AGENTS.md rules for this repo. Do NOT commit unless the user explicitly requests it. Always present the proposed commit message for review first.

## Step 6: Create the PR

IMPORTANT: Follow the git rules in the target repo's root `AGENTS.md`. Draft the branch name, commit message, PR title, and PR body, then present them to the user for review before pushing or submitting anything.

### Branch naming and push approval

Do not push until the user explicitly approves and the target repo's root `AGENTS.md` allows it.

After approval, push the branch:
```bash
git push -u origin fix/dependabot-alerts
```

### PR title

Use conventional commit format with the `chore` type:
```
chore: resolve open dependabot security alerts
```

### PR body format

Keep the PR description short. Use a single `## Summary` section with a few bullets describing the high-level changes — do NOT generate an exhaustive list or table of every alert. Dependabot will auto-close resolved alerts on merge, so the PR body does not need to enumerate them.

Aim for 1-3 bullets covering:
- The overall scope (e.g., "Resolved N of M open Dependabot security alerts by bumping vulnerable dependencies across `<area1>`, `<area2>`")
- Any notable cross-cutting changes (e.g., "Updated CI from Node 18 to Node 24", "Added overrides for transitive deps with no direct upgrade path")
- Any alerts that could NOT be resolved and why (e.g., "Alerts X, Y for `<package>` have no patched version available")

```markdown
## Summary

- Resolved N of M open Dependabot security alerts by bumping vulnerable dependencies across `<area1>` and `<area2>`
- <One-line note about any cross-cutting change, if applicable>
- <One-line note about any unresolvable alerts, if applicable>
```

Only include a "Not resolved" subsection (with alert links) when there are alerts that genuinely could not be fixed and the user should know why. Use the full URL `https://github.com/{owner}/{repo}/security/dependabot/<num>` for any alert links — never bare `#<num>`, since GitHub resolves those to issues/PRs.

Avoid:
- Per-alert bullet lists
- Resolution tables enumerating every package and version bump
- Restating what each individual `<package>` was bumped to (the diff and Dependabot itself already capture this)

### Create the PR

```bash
gh pr create --title "chore: resolve open dependabot security alerts" --body "$(cat <<'EOF'
<PR body here>
EOF
)"
```

## Step 7: Post-PR checks

1. **Monitor CI:**
   ```bash
   gh pr checks <pr_number>
   ```

2. **Common CI failure causes and fixes:**
   - **ESLint major version bump:** May need config updates or plugin compatibility fixes
   - **Python version constraints:** Patched package may require newer Python than CI matrix
   - **TypeScript compilation:** Type definition changes in updated packages
   - **Test snapshots:** May need regeneration after dependency updates

3. If CI fails, fix the issue, commit, and push again. Do NOT force-push.

## Important reminders

- Always derive `owner/repo` from `git remote -v`, never from the local folder path
- For JS transitive **dev/test deps**, bump the direct dev dependency that brings in the vulnerable package to a version that ships the fix
- For JS transitive **production deps**, ask the user before proceeding — they may prefer bumping the parent, adding a permanent override, or accepting the risk
- Some alerts may not have a patched version yet; note these as unresolvable and skip them
- If an alert is for a dev/test-only dependency, it is still worth fixing but lower priority than production dependencies
- Match the version range style used by the dependent package — if it uses `^`, your resolution should too; use an exact version only when the package is pinned exactly or has breaking changes in the patched release
- After fixing, the Dependabot alert should auto-close when GitHub detects the patched version in the default branch
