---
name: nura-bio-track-news-archive
description: >-
  Monitor Nura Bio's corporate news archive — financings, clinical milestones and journal publications —
  through the anonymous WordPress REST content API behind nurabio.com. Use when tracking Nura Bio's
  pipeline progress (NB-4746, NB-9402), funding events, or leadership announcements.
api: nura-bio:nura-bio-content-api
generated: '2026-08-26'
method: generated
source: >-
  Grounded in openapi/nura-bio-content-api-openapi.yml, derived from https://nurabio.com/wp-json/ and
  verified live 2026-08-26. Every operationId below appears in that spec.
operations:
  - listPosts
  - getPost
  - listCategories
  - getUser
---

# Track the Nura Bio news archive

Nura Bio publishes company news as WordPress posts. The archive is small and slow-moving — **9 posts**
spanning January 2020 (launch, $73M Series A) to June 2026 ($73.8M Series B) as of 2026-08-26. This is the
authoritative machine-readable source for Nura Bio announcements; there is no press API and no developer
program.

**Base URL:** `https://nurabio.com/wp-json`
**Auth:** none. Do not send credentials.

## 1. Get the current archive state

    GET /wp/v2/posts?per_page=100&orderby=date&order=desc&_fields=id,date,modified,slug,link,title,excerpt

`listPosts`. Read `X-WP-Total` from the response headers for the archive size — that single number tells you
whether anything is new since your last run, without parsing bodies.

Use `_fields` as shown. The full post body inlines rendered HTML and a large SEO metadata block; the
projection above is a fraction of the payload.

## 2. Poll for what changed

Do not re-pull the archive. Filter server-side on the modification timestamp:

    GET /wp/v2/posts?modified_after=2026-06-01T00:00:00&orderby=modified&order=desc

`modified_after` and `after` (publication date) are both ISO 8601 and both declared in the contract. Prefer
`modified_after` so you also catch corrections to existing releases, which pharma companies do issue.

## 3. Read one announcement in full

    GET /wp/v2/posts/2507

`getPost`. Returns `title.rendered`, `content.rendered` and `excerpt.rendered` as HTML strings — strip tags
before summarising. Id 2507 is the Series B release.

## 4. Separate news from careers

    GET /wp/v2/categories?per_page=100

`listCategories`. Four terms exist: **News**, **Careers**, **Uncategorized**, and one unused term. Filter
with `GET /wp/v2/posts?categories=<id>` to keep hiring posts out of a pipeline-tracking feed.

Do **not** filter by `tags` — the `post_tag` collection is registered but permanently empty on this
deployment (`X-WP-Total: 0`), so any tag filter returns nothing.

## 5. Attribute the author, if you need to

    GET /wp/v2/users/3

`getUser`. Anonymously this returns only the public projection — name, slug, description, avatar. It never
returns email or role, and the two users here are website content authors, **not** the executive team. For
leadership, use the `nura-bio-resolve-people-directory` skill.

## Conventions that apply

- **Pagination:** `page` / `per_page` (max 100, rejects above with `400 rest_invalid_param` — it does not
  clamp) / `offset`. `X-WP-Total` and `X-WP-TotalPages` are exposed cross-origin, and an RFC 8288 `Link`
  header carries `rel="next"`.
- **Errors:** the WordPress envelope `{code, message, data:{status}}`, **not** RFC 9457. Branch on `code`,
  never on `message`. An unknown id gives `404 rest_post_invalid_id`; an unknown path gives
  `404 rest_no_route`. See `errors/nura-bio-problem-types.yml`.
- **Rate limits:** none published and no `RateLimit-*` or `Retry-After` header is returned. Cloudflare bot
  management is active (`__cf_bm` is set on every response), so an aggressive poller may be challenged at
  the edge with no application-level warning. The whole archive is 9 items — poll daily at most.
- **Idempotency / reversibility:** not applicable. Every operation is a GET; the collection routes return
  `Allow: GET` anonymously. There is nothing here to undo.
