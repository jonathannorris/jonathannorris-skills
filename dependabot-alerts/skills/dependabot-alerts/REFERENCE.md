## Get detailed info for a specific alert

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

## npm / yarn / pnpm

1. Trace the dependency tree:
   ```bash
   yarn why <package>    # yarn
   npm ls <package>      # npm
   pnpm why <package>    # pnpm
   ```

2. **Direct dependency:** Bump in `package.json` `dependencies` or `devDependencies`.

3. **Transitive — try lockfile-only bump first (preferred):**
   ```bash
   yarn upgrade <package>              # yarn v1
   yarn up "<package>@^<patched>"      # yarn v2/v3/v4
   npm update <package>               # npm
   pnpm update <package>              # pnpm
   ```
   Verify: `yarn why <package>` / `npm ls <package>`

4. **If lockfile-only doesn't work:**
   - **Dev/test transitive deps:** Bump the direct dev dep that ships the patched transitive version
     ```bash
     npm view "<parent>@latest" dependencies
     ```
   - **Production transitive deps:** Ask the user — options are bumping the parent, adding a permanent override, or accepting the risk

5. **When overrides are necessary** (patched version outside all existing semver ranges):
   - **yarn v1:** `"resolutions": { "<package>": "^<patched>" }` in `package.json`
   - **npm:** `"overrides": { "<package>": "^<patched>" }` in `package.json`
   - **pnpm:** `"pnpm": { "overrides": { "<package>": "^<patched>" } }` in `package.json`

   **npm lockfile-pinning trick** (avoid leaving permanent overrides):
   1. Add override to `package.json`
   2. Run `npm install`
   3. Remove override from `package.json`
   4. Run `npm install` again
   5. Verify with `npm ls <package>`

6. Regenerate lockfile: `yarn install` / `npm install` / `pnpm install`

7. **Monorepos:** Check all `package.json` files — each workspace may need its own resolution.

## pip (Python)

1. Identify the manifest from the alert's `manifest` field
2. Bump the version pin in the relevant requirements file
3. Check Python version constraints — if the patched version requires a newer Python than CI supports, consider splitting requirements files or adjusting CI Python versions
4. Check `pyproject.toml` and `setup.cfg` for constraints that also need updating

## Go

```bash
go get <module>@latest
go mod tidy
```

Verify `go.mod` and `go.sum` resolve to the patched version, then `go build ./...` and `go test ./...`.

## Java / Gradle / Maven

**Gradle:** Update versions in `build.gradle` / `build.gradle.kts`. For transitive overrides:
```groovy
configurations.all {
    resolutionStrategy {
        force 'group:artifact:version'
    }
}
```

**Maven:** Update in `pom.xml` or use `<dependencyManagement>`. Verify with `mvn verify`.

## Swift

Update pinned versions in `Package.resolved`. Run `swift package resolve` and `swift build`.

## .NET / NuGet

Update package versions in `.csproj` files or `Directory.Packages.props`. Run `dotnet restore` and `dotnet build`.

## Ruby / Bundler

Update version constraints in `Gemfile`. Run `bundle update <gem>` and confirm `Gemfile.lock` resolves to the patched version.

## GitHub Actions

Bump action versions in `.github/workflows/*.yml`:
```yaml
# Before: uses: actions/checkout@v4
# After:  uses: actions/checkout@v5
```

## Key reminders

- Always derive `owner/repo` from `git remote -v`, never from the local folder path
- For JS transitive deps, always try lockfile-only first; only add `resolutions`/`overrides` when the patched version is outside all existing semver ranges
- For JS dev/test transitive deps: bump the direct dev dep instead of using overrides
- For JS production transitive deps: ask the user before adding permanent overrides
- Some alerts may have no patched version yet; note these as unresolvable and skip them
- Match the version range style of the dependent package (`^` if it uses `^`)
- Do NOT use `#<number>` in commit messages or PR bodies for alert references
