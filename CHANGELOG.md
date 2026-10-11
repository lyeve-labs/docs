# Changelog

All notable changes to the LyEve documentation are recorded here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and the repository is versioned as
[VERSIONING.md](VERSIONING.md) describes.

## [0.2.0] - 2026-10-11

### Added

- Every feature page with a screen in the admin console says where it is, what it shows and how to use it there, with a screenshot.

### Fixed

- The API key pages said access rules and a key's list of content types do not limit a key. A key's roles go through the same access rules as a person's, field masks included, and a key is refused every content type outside its list. The walkthrough now gives its key a role with a read rule first.

## [0.1.0] - 2026-10-11

### Added

- The LyEve documentation, published at lyeve.com/docs: getting started, the concepts, a usage
  guide for every feature, how-to recipes, integrations, the client libraries, deployment and
  scaling, the reference and the trust pages.
- A `redirects` file that keeps every old documentation address answering.
- How to report a mistake with an issue, or fix it with a pull request into `dev`.
