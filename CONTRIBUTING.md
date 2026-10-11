# Contributing to LyEve Documentation

Thank you for improving the docs. A page is held to the same standard as code: every fact on
it is true of the product today.

## Ground rules

1. **One source.** Public documentation lives here, in `docs/`. lyeve.com/docs is built from
   each tagged release of this repository, so a merged change goes live with the next release.
2. **One home per fact.** Say a thing once, on the page that owns it, and link to it from
   everywhere else. A second copy is the copy that goes stale.
3. **Accuracy over completeness.** Never invent a command, flag, route, setting or default.
   Check each one against the product's source or a running instance. If a fact cannot be
   confirmed, leave it out and open an issue. Do not park a `TODO` in an HTML comment: comments
   ship to the browser and are readable in view-source.
4. **Usage, not internals.** Write how to use LyEve. Leave out how it is built: source paths,
   package and type names, database tables, boot order, and the history of renames.
5. **Voice.** Plain, direct and specific. US English, short sentences, one idea each. No hype,
   no emoji, no filler. Name something a reader can check instead of saying how good it is.

## Page shape

Every page starts with front matter:

```yaml
---
title: Concise Title
description: One sentence, under 160 characters, starting with a verb.
sidebar:
  order: 3
---
```

A feature page lives in `docs/features/`, is named by what the reader does, and follows this
order. `docs/features/flows.md` is the model.

1. The requirement line, first after the heading: `> **Included free** on every install.` or
   ``> **Requires** a license with the `<feature>` feature.``
2. Two to four sentences on what the feature does for the reader.
3. A `:::tip[Best for]` box with three to five concrete jobs, and a
   `:::note[Use something else when]` box when another feature fits a neighboring job better.
4. **Try it**: a few numbered steps a reader can paste and run, with the expected result.
5. Sections named by the task ("Retry a failed delivery"), then **Settings**.
6. Reference a reader looks up rather than reads, such as route tables and long payloads,
   inside a `<details>` block with a `<summary>`. Leave a blank line after the summary and
   before `</details>`, or the Markdown inside does not render.
7. **Related**: a few links, each saying why.

How-to pages: the goal, **Before you start**, numbered steps with runnable commands and
expected output, then **Next**.

The site renders the Starlight callouts `:::tip`, `:::note`, `:::caution` and `:::danger`, with
an optional title in brackets. Use them for guidance and real limits, not decoration. No other
HTML, and no MDX.

## Structure

- One subject per page. A page that answers two questions becomes two pages.
- Tables for reference material: settings, routes, status codes, comparisons.
- Relative Markdown links between pages (`../features/webhooks.md`), so they work on GitHub
  and on the site.
- Every command runs as written, including the auth header and the content type.
- A page that moves or goes away gets a line in `redirects` naming the page that holds its
  subject now, so an old link answers 301 rather than 404.
- A page in an existing directory appears in the navigation on its own. A new directory needs
  a group in the site's navigation, which a maintainer adds when the page is released.

## Screenshots

- A screenshot lives in the `images/` directory beside the page that shows it, and the page
  links it with a relative path (`![What the screen shows](images/entry-list.png)`). The site
  resizes and converts it at build, and GitHub renders the same link. Never link an image hosted
  somewhere else, such as a raw GitHub URL: a host can require a sign-in, rate-limit readers or
  drop the file, and the build cannot see any of it.
- The alt text says what the screen shows, so the page still reads without the picture.
- Screenshots are captured from a fresh instance with demo data, never edited by hand, and
  the maintainers retake them after a console change. If one no longer matches the console,
  open an issue naming the page.
- A screenshot shows demo data only: no real accounts, keys, tenants or license tokens.

## Workflow

`dev` is integration and `main` carries the release tags. Both take merges only. Fork the
repository, branch from `dev` and open the pull request back into `dev`. A question, or a
mistake you cannot fix yourself, is an issue.

```bash
git checkout dev && git pull
git checkout -b docs/<slug>
# edit Markdown under docs/
git commit -m "docs(<area>): <what changed>"
```

Commit subjects follow [Conventional Commits](https://www.conventionalcommits.org/).
`.githooks/commit-msg` checks the subject. Copy it into `.git/hooks/` to run the check on every
commit. Do not add authorship or tooling trailers.

## Versioning

Every release is tagged and logged. See [VERSIONING.md](VERSIONING.md).
