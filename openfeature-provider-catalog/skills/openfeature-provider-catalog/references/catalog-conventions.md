# Catalog Conventions

Use the repository's existing AGENTS.md instructions for Git, worktrees, commits, and GitHub operations.

Provider records live in `src/datasets/providers/` and implement the `Provider` type from `src/datasets/providers/index.ts`.

```ts
import VendorSvg from '@site/static/img/vendor-no-fill.svg';
import type { Provider } from '.';

export const Vendor: Provider = {
  name: 'Vendor',
  logo: VendorSvg,
  technologies: [
    {
      technology: 'JavaScript',
      vendorOfficial: true,
      href: 'https://vendor.example/openfeature',
      category: ['Server'],
    },
  ],
};
```

## Technology mapping

- Node.js and browser providers use `technology: 'JavaScript'`, with `category: ['Server']` for Node.js and `category: ['Client']` for browser providers.
- Android providers use `technology: 'Kotlin'` because that is the catalog's Android technology label.
- iOS and Apple-platform providers use `technology: 'Swift'` and usually `category: ['Client']`.
- Go, Java, Python, Ruby, PHP, and `.NET` providers generally use `category: ['Server']` unless primary sources establish a client SDK/provider.
- React, Angular, NestJS, Next.js, SvelteKit, and other framework values require an actual framework-specific provider, not merely compatibility with a base provider.
- Use only values in the `Technology` union in `src/datasets/types.ts`.

## Links and official status

- Prefer the vendor's provider documentation page over a package registry or repository when it clearly documents setup.
- Use the provider repository when documentation is absent or the repository is the canonical source.
- Mark `vendorOfficial: true` when the vendor owns or maintains the provider. Use `false` for OpenFeature-contrib or other community implementations.
- Verify every URL with a request or source fetch before adding it.
- Keep one provider record per vendor, with one technology entry per supported provider and category.

## Edge cases

- A normal vendor SDK is not an OpenFeature provider by itself.
- If a provider supports both browser and Node.js, add two JavaScript entries with separate links and categories.
- If a single documentation page covers multiple provider platforms, it may be reused for those entries.
- Do not add a platform that is only mentioned in a roadmap, example application, or regular SDK documentation.
