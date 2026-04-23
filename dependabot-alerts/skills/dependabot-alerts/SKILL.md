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

3. **Transitive dependency:** Use resolutions/overrides to force the patched version. First run `yarn why <package>` (or `npm ls`/`pnpm why`) to see what range the parent uses, then **match that range style** in your resolution:
   - If the parent specifies `^4.12.12`, use `^<patched_version>` (e.g. `^4.12.14`)
   - If the parent specifies an exact version or the package has no semver-compatible patched release, use an exact version (e.g. `4.12.14`)
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

Use conventional commit format:

```
fix: resolve open dependabot security alerts
```

For commits addressing multiple alerts, include a body listing the changes:

```
fix: resolve open dependabot security alerts

- <package> <old_version> -> <new_version> (<severity>, alert #<number>)
- <package> <old_version> -> <new_version> (<severity>, alert #<number>)
```

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

Use conventional commit format:
```
fix: resolve open dependabot security alerts
```

### PR body format

**For fewer than 3 alerts**, use a simple bullet list:

```markdown
## Summary

- Bumped <package> to <version> to resolve <severity> vulnerability (<summary>)
- Added resolution for <package> to fix transitive <severity> vulnerability
```

**For 3 or more alerts**, include a resolution table:

```markdown
## Summary

- Resolved N open Dependabot security alerts by bumping vulnerable dependencies

## Dependabot Alerts Resolved

| Alert | Package | Severity | Fix |
|-------|---------|----------|-----|
| #<num> | `<package>` | **<severity>** | Bumped to <version> via <mechanism> |
```

### Create the PR

```bash
gh pr create --title "fix: resolve open dependabot security alerts" --body "$(cat <<'EOF'
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
- For JS transitive deps, prefer `resolutions`/`overrides` over upgrading the parent package (less risk of breaking changes)
- Some alerts may not have a patched version yet; note these as unresolvable and skip them
- If an alert is for a dev/test-only dependency, it is still worth fixing but lower priority than production dependencies
- Match the version range style used by the dependent package — if it uses `^`, your resolution should too; use an exact version only when the package is pinned exactly or has breaking changes in the patched release
- After fixing, the Dependabot alert should auto-close when GitHub detects the patched version in the default branch
