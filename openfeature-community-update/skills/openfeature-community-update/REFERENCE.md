# Reference

## Worktree setup

Worktrees live in `.worktrees/` inside the repo (gitignored). The main clone stays untouched.

```bash
mkdir -p ~/git/OpenFeature/openfeature.dev/.worktrees
git -C ~/git/OpenFeature/openfeature.dev pull
git -C ~/git/OpenFeature/openfeature.dev worktree add .worktrees/<name> -b blog/<name>
```

Run every command from the worktree. A worktree-isolated session rejects compound shell (`for` loops, redirects, process substitution) as "too complex to verify that it stays inside the worktree", so prefer plain single commands, `xargs`, or one batched `gh api graphql` call.

The git stash stack is shared across worktrees. Use a WIP commit instead of `git stash`.

## Proven section structure

Ordered as it shipped in the mid-2026 update:

1. **Two-line intro** naming the headline items, then `<!--truncate-->`
2. **Platform-native feature flags** -- large platforms that launched flag products supporting OpenFeature, one bullet each in launch order, each with what they shipped and how they support OpenFeature
3. **New vendors in the ecosystem** -- two lists: companies shipping their first providers (alphabetical), then existing vendors expanding language coverage. Close with the "if your company has a provider that is not listed" CTA linking the provider issue template and `/ecosystem`
4. **Governance or Technical Committee changes** -- new members, with what they work on and their contribution history
5. **Specification release** -- the headline stabilization PR, then the accumulated features, then breaking changes called out separately, then first-time contributor count
6. **Also shipped** -- one bullet per theme (not per PR), each linking every relevant PR. Cross-SDK rollouts state which SDKs released vs merged-awaiting-release
7. **Where we could use help** -- larger cross-cutting projects only, never one-off SDK issues. Each item names the cross-language consequence of it staying unfinished, and links a per-SDK tracking issue where one exists
8. **Closing** -- Slack, community meetings, GitHub, next KubeCon

## Vendor-neutral voice rules

| Do | Do not |
| --- | --- |
| "launched in late 2025 and describes its product as 'built on the OpenFeature standard'" | "the most complete OpenFeature integration" |
| "Five well-known platforms launched feature flag services of their own" | "the biggest vendors have finally caught up" |
| "its docs scope it to server-side use, with their own SDK covering client-side" | "OpenFeature is their primary SDK interface" (verify this before claiming it; it is usually false) |
| State the ordering: launch order, alphabetical | Imply a ranking through ordering |
| "now maintains seven official providers in beta under its own organization" | "has the broadest provider set" |

Vendors are singular entities: "each chose OpenFeature as a critical part of **its** product offering".

Check whether a platform is genuinely built on OpenFeature or merely wraps a native implementation. Read the provider README: wording like "provides a bridge between X's native feature flags implementation and the OpenFeature specification" means native-first with an OpenFeature wrapper, which is a different claim.

## Verification recipes

### Batch PR and issue states

One call, many refs, avoids rate limits:

```bash
gh api graphql -f query='{
  a: repository(owner:"open-feature",name:"java-sdk"){ pullRequest(number:1985){ title state mergedAt } }
  b: repository(owner:"open-feature",name:"kotlin-sdk"){ pullRequest(number:225){ title state mergedAt } }
  c: repository(owner:"open-feature",name:"spec"){ issue(number:417){ title state } }
}'
```

### Releases

`gh release list` may be blocked; use the API:

```bash
gh api repos/open-feature/flagd/releases --jq '.[0:5][] | "\(.tag_name) \(.published_at)"'
```

flagd uses prefixed tags, so links need URL encoding: `.../releases/tag/flagd%2Fv0.16.0`.

### Provider inventory in an org

```bash
gh api "search/repositories?q=org:Unleash+openfeature&per_page=30" --jq '.items[].full_name'
```

### Provider directories inside contrib repos

```bash
gh api graphql -f query='{
  js: repository(owner:"open-feature",name:"js-sdk-contrib"){ object(expression:"main:libs/providers"){ ... on Tree { entries { name } } } }
}'
```

### Links

- GitHub URLs: verify with `gh api repos/<owner>/<repo>/contents/<path>`, not `curl`. Unauthenticated parallel curl gets 503/000 from rate limiting, and repos get renamed (a 200 may be a redirect to a new name; use the canonical URL).
- External URLs: `curl -s -o /dev/null -w "%{http_code}\n" <url>`. npm returns 403 to bots, which is not a dead link. Watch for domain-parking pages returning 200; check the `<title>`.
- Internal routes: confirm the file exists under `docs/`, then rely on the Netlify build. `/docs/reference/technologies/*` is a redirect alias, the real path is `/docs/reference/sdks/*`. flagd has no docs route, link `https://flagd.dev/`.

## Publish checklist

- [ ] No em dashes: `grep -c "—" blog/<file>.md` returns 0
- [ ] One sentence per line
- [ ] Every count in the prose matches the number of bullets below it, including the frontmatter `description`
- [ ] Frontmatter `description` avoids naming individual companies, so it does not go stale
- [ ] Signed off: `git commit -s`, conventional prefix (`docs:`), single summary line, no AI trailers or co-author lines
- [ ] Draft PR: `gh pr create --draft`. PR body passed directly to `--body`, never via heredoc
- [ ] Netlify check passed, which validates internal links via `onBrokenLinks: 'throw'`
- [ ] `gh pr ready <n>` only when the author asks

If running markdownlint locally, note `fix: true` in the config rewrites unrelated files under `docs/reference/other-technologies/ofrep/`. Revert them with `git checkout --` so only the post stays changed.

## Hero image

`static/img/blog/<date>-<slug>/` with the `image:` frontmatter key. Leave `image:` commented out rather than pointing at a file that does not exist.
