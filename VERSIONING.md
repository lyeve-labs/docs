# Versioning and Releases

## 1. Every release is versioned

Changes land on `dev` through pull requests. A release merges `dev` into `main` and tags it.
Every release that changes published documentation is a new SemVer tag with its
`CHANGELOG.md` entry in the same commit. `main` carries no untagged content change.

| Change | Commit type | Version bump | CHANGELOG section |
|---|---|---|---|
| Correction, typo, broken link | `fix:` | **PATCH** (`x.y.Z`) | `### Fixed` |
| New page or section | `feat:` or `docs:` | **MINOR** (`x.Y.0`) | `### Added` |
| Restructure that moves or removes published URLs | `feat!:` or `BREAKING CHANGE:` | **MAJOR** (`X.0.0`, MINOR while `0.x`) | `### Changed` or `### Removed` |
| Copy edits, refactors, chores | those types | released with the next bump | `### Changed` |

While the version is `0.x`, a breaking change bumps MINOR, not MAJOR (SemVer section 4). The
repository moves to `1.0.0` when the public URL structure is declared stable. A moved page
always gets a line in `redirects`, so an old link keeps answering.

## 2. Workflow for a change

```bash
git checkout dev && git pull
git checkout -b docs/<slug>
# edit Markdown
git commit -m "docs(deploy): add the Neon guide"
# open a pull request into dev, then review and merge
```

A maintainer cuts each release: the version follows from the commits, the CHANGELOG entry is
written, and `main` gets an annotated tag. Never tag by hand from a working branch.

## 3. Commit checks

`.githooks/commit-msg` rejects a subject that is not a Conventional Commit. Copy it into
`.git/hooks/` to run it on every commit. Do not add authorship or tooling trailers.

## 4. Tag format

- `vMAJOR.MINOR.PATCH`, annotated tags only.
- A published tag never moves. The documentation site pins a docs version by tag, and a bad
  release is replaced by the next patch.

## 5. CHANGELOG

`CHANGELOG.md` follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), newest version
first, with one header per version: `## [X.Y.Z] - YYYY-MM-DD`. Group changes under `Added`,
`Changed`, `Fixed`, then `Removed`, `Deprecated` or `Security` as needed. One line per change,
imperative and plain.
