# Changelog

All notable changes to the LyEve documentation are recorded here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and the repository is versioned as
[VERSIONING.md](VERSIONING.md) describes.

## [0.3.1] - 2026-10-12

### Fixed

- The roles and permissions guide said access rules do not apply to API keys. A key is judged by the roles it carries, so a key with no role reads no content, and the guide says so. The troubleshooting table names the 403 a key with no role meets, and how to fix it.

### Changed

- Page titles are written in sentence case, so the sidebar, the page headings and the search results read the same way as the rest of the site. "Your First Content Type" is now "Your first content type", and an ampersand in a title is now "and".

## [0.3.0] - 2026-10-12

### Added

- The API key pages describe scopes that hold a key to one content type or one endpoint, the route that lists every scope an instance offers, and hourly and daily request limits beside the monthly one.
- The backup guide says how archives are encrypted with AWS KMS, where the credentials come from, and which permissions the key needs.

### Fixed

- The API key pages said a key's list of content types was not enforced. A key is refused every content type outside its list, and the pages now say so and warn that GraphQL does not honor the list yet.
- The free hosting guides tell the host's start apart from the engine's in a cold start, and name the log line that measures the engine's share.
- The data export page says an S3 export lands under the tenant's own prefix, and lists the answer for a key that tries to leave it.
- The bulk import page states each size limit and the answer it gives.
- The source code page put content types, entries, draft and publish and revisions under the MIT engine. They come from free features that ship as commercial software, and the page now lists them there.
- The deploy overview said the admin console's source is not published. It is public, and the console runs as its own service on any host that runs containers.

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
