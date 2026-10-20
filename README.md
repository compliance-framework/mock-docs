# mock-docs

The [CCF Factory](https://github.com/compliance-framework/factory)'s safe target: a mirror of [compliance-framework/docs](https://github.com/compliance-framework/docs) that the factory's **dev** RuleSets work on (mock release trains, fake versions, the i18n epic) before anything touches the real docs.

Nothing here is authoritative. For the real documentation, see https://compliance-framework.github.io/docs/.

## What's in it

- **The mirror.** Everything except this README is copied from `compliance-framework/docs` at branch `fx/doc-s1-t05-catalog`, the top of the docs revamp stack, by factory task DOC-S2-T01. The real instances in `instances.yaml` stay, so the factory sees the real layout. [STYLE.md](STYLE.md) and [CONTRIBUTING.md](CONTRIBUTING.md) say how pages are written and versioned.
- **Mock instances.** `instances.yaml` adds instances for the mock repositories, with placeholder pages marked as mock content and fake version lines `0.0` and `0.1`:
  - the products `mock-api`, `mock-ui` and `mock-agent`, in the navbar;
  - the plugins `mock-plugin-1` and `mock-plugin-2`, in the catalog, with the policy bundles `mock-plugin-policies-1` and `mock-plugin-policies-2`.
- **Its own site.** Both workflows set `DOCS_PROJECT=mock-docs`, so the site builds under `/mock-docs/` and every push to `main` deploys it to this repository's GitHub Pages, at https://compliance-framework.github.io/mock-docs/.

## Build it

You need Node.js 20 (see [`.nvmrc`](.nvmrc)) and Yarn 1.

```sh
yarn install --frozen-lockfile
yarn test
DOCS_PROJECT=mock-docs yarn build
DOCS_PROJECT=mock-docs yarn serve
```

Without `DOCS_PROJECT`, the site builds under `/docs/`, like the real one.
