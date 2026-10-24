# Changelog

All notable changes to the LyEve documentation are recorded here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and the repository is versioned as
[VERSIONING.md](VERSIONING.md) describes.

## [0.6.1] - 2026-10-24

### Changed

- The source code page limits the admin console's MIT entry to its public source, and names the plugin license agreement on lyeve.com as the terms for license verification, for the visual flow and schema editors and for any other LyEve code in the images whose source is not public. The license file the admin image ships for the editors is described as a summary of that agreement.

## [0.6.0] - 2026-10-23

### Added

- Every page that shows a task offers two ways to follow it: in the admin console or with the engine's API. The site shows them as two tabs labeled with the releases the steps describe, admin console v0.21.0 and engine v0.55.0, and a task only one of them can do says which. Every step on both paths was run against those releases, the licensed parts on a licensed install.
- `versions.json` at the root names the engine and console releases the pages describe, and the contributing guide says how to write a task with both paths.
- The AI page shows how to suggest alt text for an image, from the media drawer or with the API.
- The paid parts of free pages offer the console steps beside the API calls too, run on a licensed install: among them a review stage that needs two approvals, a provider for any HTTP mail API, tenant captcha settings, and a webhook's retries and dead letters.

### Changed

- The deploy and configuration pages say that a restart ends every session once its access token expires, unless `CACHE_DRIVER=redis` and `REDIS_URL` point at a Redis. A volume that keeps the signing key is not enough on its own.
- The realtime page no longer offers a browser example for server-sent events, which the engine refuses from the instance's own address. A browser uses the WebSocket, and a server uses the event stream.
- The CI gate in the guide to schema and flows as code signs in from stored secrets, because the plan route takes no API key and a session token lasts 15 minutes.

### Fixed

- The search page says a snippet is HTML with its text escaped and each match in mark tags, and every other field of a hit is plain text.
- The data export page says to keep names and email addresses out of an export's S3 key, which outlives an erasure.
- Getting started: setup asks for the password twice, the console does not fill in a content type's machine name, and only the API turns on drafts and publishing.
- An MFA code cannot be used twice at sign-in, the example password meets the default policy, and a new sign-in provider starts with Enabled on in the console.
- The custom API guide publishes its flow from a tenant account, because the first administrator belongs to no tenant and its flow stopped at the first content read.
- The download of an export sent to S3, the tenant the quota lists show, the units of the quota form and how public search types are chosen are corrected.
- A SAML sign-in through a form post completes, a directory sending the SCIM media type is accepted, a streamed audit entry can take about 30 seconds, and the audit log example filters on an action that is recorded.
- The security controls page names the events the audit trail records on engine v0.55.0 and the account and sign-in events it does not.
- Deleting a tenant removes its media records and leaves the stored files in the storage backend. A capability name cannot be withheld from a tenant, and a move between tenants needs the content types in place first.
- A tenant with no API key reads its usage, a legal hold keeps an audit prune from removing the entries it covers, and the object storage page names the headers a direct upload must send.
- Saved firewall settings survive a restart, the firewall skips the `sql` field of a backup validate or import, and two content types that refer to each other import in one apply.
- Slow query analysis lists requests with no tenant and shows the statements with the highest average time since the statistics were last reset, and installing the PostgreSQL extension needs a restart.
- The AI settings sit under Settings > Configuration, the common gRPC command line client cannot call the content services, a NATS server without TLS needs `NATS_TLS=false`, and on engine v0.55.0 a broker message carries the zero time.
- The event replay prune names the tenant, because a super admin's prune that names none clears every tenant's events.
- The two-factor page no longer says a passkey can finish a password sign-in. A password sign-in asks for an authenticator code, so an account whose only factor is a passkey cannot sign in to the console, and the page says to add an authenticator app first.
- A webhook update replaces the whole webhook, so a field left out is cleared, and a retry policy set with the API wins over the webhook's own retry numbers.
- On a review stage that needs several approvals, only an assignee may approve.
- The console's New user form keeps editor ticked beside any role you tick, so the pages that create an account with another role say to untick it.

## [0.5.0] - 2026-10-20

### Added

- Eight pages carry a diagram beside the text it explains: how the quickstart stack routes requests, which API a caller uses, how an entry moves between draft, published and archived, how a flow runs, what happens to a webhook delivery, how a request picks its tenant, what replicas share, and how a service verifies a token. Each one follows the site's light or dark theme and says where a step needs a license.

### Fixed

- The tenants page said an `X-Tenant-ID` header from anyone but a super admin is ignored. An admin token that names another tenant is refused with a 403.
- The webhooks page gives the 30 seconds a retry or a replay has to answer, and says that replaying or dismissing a dead letter needs `webhook-pro`.
- The scaling page names the response cache, says that a job saved on another replica reaches the schedule within about a second only with replication, and says that broker delivery is at least once.
- The token validation page said a service stops accepting tokens signed with the old key as soon as the key is rotated. A service keeps accepting such a token until it expires or the service fetches the key set again.

## [0.4.0] - 2026-10-20

### Added

- The pages describe engine 0.55.0 and console 0.21.0: draft and access rules on every content transport, proof before a second factor is added or removed, accounts a tenant's identity provider creates and how long they may sign in without it, audit chain checks with scheduled retention, backup schedules, webhook retries, caching, and the outbound address guard with the setting that opens the shared address space to outbound requests.
- The command reference covers `lyeve migrate mark-applied`.
- A super admin with no tenant picked can search, tail and measure the logs, and the pages say where a super admin's read of personal data is filed and that subject export and erasure audit entries go under no tenant.
- Request capture replay goes only to the engine's own API or admin listener and does not follow redirects, so credentials reach no other host. A super admin's replay of a capture that belongs to no tenant answers 422.
- The feature pages list the new refusals and limits: bulk import into another tenant when the database pool holds fewer than four connections, a storage key with a `.` segment, a probe that names a header twice or sends the mask, `$env` in a flow expression, a closed region as the default, and a SCIM deprovision action other than disable or delete. They also cover flow datasource secret updates and secret headers, the MFA verify 409, the limit of 32 GraphQL subscriptions, the object key each audit retention archive is written under, the pricing link on a free ceiling refusal, and that an update never changes an entry's status.

### Changed

- The deployment guides pin engine 0.55.0 and console 0.21.0 and upgrade the engine first, because console 0.21.0 needs engine 0.55.0 or later. The Compose and Kubernetes steps start the engine and wait for it before they move the console, and the upgrade notes list the changes API clients see and the new error alerts.
- The Kubernetes guide routes with the Gateway API instead of an Ingress controller that Kubernetes retired in March 2026. It names what the routing layer must provide: a request body limit of at least 68 MiB, a read timeout long enough for a large backup import, and one console replica unless the gateway keeps each browser on one replica.
- The Kubernetes guide runs one console replica, because two replicas with no session affinity sign users out when both renew the same session at once. Cookie affinity at the gateway allows more.

### Fixed

- The security page called the reason for a second factor reset optional. The reset requires one.
- The configuration reference, the media settings and the tenant backup guide said the general body limits apply to every route. A route that sets its own limit keeps it, so bulk import accepts close to 68 MiB and tenant backup import up to 500 MiB, and a chunked request is cut off at the same size as any other.
- The content lifecycle errors table lists the 404 answers for scheduling an entry that does not exist.
- The PII masking page names its one exception: SCIM provisioning answers are left as written, so an identity provider reads back the people it provisioned, and those reads are not in the access log.
- The message broker status table says what the connected field means for each backend. For Kafka it says whether a broker answered within two seconds, and NATS and RabbitMQ keep the old meaning.

## [0.3.3] - 2026-10-15

### Fixed

- Links to the pricing, contact and plugin license pages named each address without its trailing slash, so every one went through a permanent redirect. They now name the address the site serves.
- The flow catalog reference page showed a second top-level heading under its title. It is now a section heading.
- The analytics, API keys, data protection and security controls pages carried descriptions longer than a search result shows. They are shorter and keep the same facts.

## [0.3.2] - 2026-10-14

### Added

- The Docker deployment page describes release candidates: the image tag each one carries, that a candidate never moves the latest or minor tag, how to run one, and that a database which ran a candidate cannot go back to an earlier release.

### Fixed

- The installation and deployment pages gave a MySQL connection URL that starts the engine but fails the first setup request. The URL now carries `parseTime=true`, and a new section says the database needs the `utf8mb4_unicode_ci` collation, which a stock MySQL server or container does not give it.
- The pages said any MySQL from version 8 works. They now name the tested versions, 8.0 and 8.4, and say that MySQL 9 is not supported yet.

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
