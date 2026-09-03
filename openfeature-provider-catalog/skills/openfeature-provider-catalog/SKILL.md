---
name: openfeature-provider-catalog
description: Add OpenFeature provider entries to openfeature.dev by researching provider documentation, classifying technologies, and adding compliant catalog logos. Use when adding a provider, vendor SDK, provider link, or ecosystem entry to openfeature.dev.
---

## When to use this skill

- Add a vendor's OpenFeature providers to `openfeature.dev`.
- Update provider languages, official status, documentation links, or logos.
- Turn a vendor announcement or provider list into a catalog PR.

## Workflow

1. Research the vendor announcement and provider documentation. Confirm each supported platform, client/server category, canonical URL, package or repository, and whether the vendor maintains the provider. A regular SDK is not automatically an OpenFeature provider.
2. Inspect `src/datasets/providers/index.ts`, similar provider files, `src/datasets/types.ts`, and existing logo assets.
3. Add `src/datasets/providers/<vendor>.ts`, import it in `index.ts`, and register it in `PROVIDERS`. Use one provider record with one technology entry per supported provider and category.
4. Add `static/img/<vendor>-no-fill.svg` using recognizable geometry only. See [SVG rules](references/svg-rules.md).
5. Run focused validation: `git diff --check`, ESLint on changed TypeScript files, and the SVG paint-declaration check. Run the repository typecheck when dependencies are available and separate baseline failures from new failures.
6. Follow the repository's established Git and GitHub workflow for committing, pushing, and opening a draft PR. Link the primary vendor source and report pending CI separately.

## Catalog conventions

See [catalog conventions](references/catalog-conventions.md) for technology mapping, provider categories, official status, links, and edge cases.

## Guardrails

- Never silently overwrite unrelated user changes.
- Do not add a separate provider for a platform unless primary sources show an OpenFeature provider exists.
