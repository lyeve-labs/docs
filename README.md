# LyEve Documentation

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

The public documentation for [LyEve](https://lyeve.com), a self-hosted headless CMS written in
Go. _All under your control._

Read it at [lyeve.com/docs](https://lyeve.com/docs/). This repository is its single source:
every page is plain Markdown with front matter, the site is built from each tagged release of
`docs/`, and the pages render on GitHub as they are.

The pages say how to use LyEve, not how it is built. Each feature page opens with a line that
says whether the feature is included free or needs a license, then what it is best for and a
short "Try it" you can run against your own instance.

## What's inside

| Section | What it covers |
|---|---|
| [Getting Started](docs/getting-started/index.md) | What LyEve is, installation, the quickstart, your first content types |
| [Concepts](docs/concepts/index.md) | Content types and entries, tenants, roles, the two APIs, licensing |
| [Features](docs/features/index.md) | One usage guide per feature, free and licensed, with a catalog by job |
| [Guides](docs/guides/) | A path for each reader: editors, developers, operators, evaluators |
| [How-To](docs/how-to/index.md) | Step-by-step recipes against a running instance |
| [Integrations](docs/integrations/index.md) | Choosing how to connect other systems, sign-in options, verifying tokens |
| [SDK](docs/sdk/index.md) | The client libraries |
| [Deploy](docs/deploy/index.md) and [Scaling](docs/scaling/index.md) | Where to run it, every platform guide, the production checklist, tuning |
| [Reference](docs/reference/configuration.md) | Configuration, API endpoints, CLI, the flow definition format, glossary |
| [Trust](docs/trust/source-code.md) | Security controls, data protection, source code, offline license checks |

## Report or fix a mistake

Found a step that does not work, a setting that is wrong, or a page that is unclear? Open an
issue naming the page and what you saw. Better still, fix it: every page is a Markdown file
under `docs/`. Fork this repository, branch from `dev`, edit, and open a pull request back into
`dev`:

```bash
git clone https://github.com/<you>/docs.git
cd docs
git checkout dev
git checkout -b docs/<slug>
# edit docs/getting-started/quickstart.md
git commit -m "docs(getting-started): clarify the sign-in step"
git push origin docs/<slug>
```

Commit subjects follow [Conventional Commits](https://www.conventionalcommits.org/).
`.githooks/commit-msg` checks the subject. Copy it into `.git/hooks/` to run the check on every
commit. [CONTRIBUTING.md](CONTRIBUTING.md) has the page shape, the voice and the rules for
checking facts.

## Preview

The pages render on GitHub as they are. The published site adds the callout boxes, the folded
reference sections and the navigation. A merged change reaches lyeve.com/docs with the next
release.

## Repository map

```text
docs/
├── docs/                  published to lyeve.com/docs
│   ├── index.md           landing page
│   ├── getting-started/   what LyEve is, installation, quickstart, first content types
│   ├── concepts/          the ideas the other pages assume, data model, licensing
│   ├── features/          one usage guide per feature, and the catalog
│   ├── guides/            editors, developers, operators, evaluators
│   ├── how-to/            task recipes
│   ├── integrations/      choosing a connection, sign-in options, JWKS
│   ├── sdk/               the client libraries
│   ├── deploy/            where to run it, every platform, the production checklist
│   ├── scaling/           replicas, Redis, the pool, caching, shutdown
│   ├── reference/         configuration, API endpoints, CLI, flows, glossary
│   └── trust/             security, data protection, source code, offline verification
├── redirects              every moved path and the page that holds its subject now
├── CHANGELOG.md           Keep a Changelog, newest first
├── CONTRIBUTING.md        page shape, voice, workflow
├── VERSIONING.md          how this repository is versioned
└── LICENSE                MIT
```

## Versioning

Every release of `main` is a SemVer tag with its CHANGELOG entry. See
[VERSIONING.md](VERSIONING.md).

## License

MIT. See [LICENSE](LICENSE).
