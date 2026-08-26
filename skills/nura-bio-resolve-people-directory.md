---
name: nura-bio-resolve-people-directory
description: >-
  Resolve Nura Bio's leadership team, board of directors, and founders and advisors from the anonymous
  WordPress REST content API. Use when identifying who runs Nura Bio, who sits on its board, or who its
  scientific founders are — and note the directory lives on the `portfolio` post type, not on `users`.
api: nura-bio:nura-bio-content-api
generated: '2026-08-26'
method: generated
source: >-
  Grounded in openapi/nura-bio-content-api-openapi.yml and data-model/nura-bio-data-model.yml, derived from
  https://nurabio.com/wp-json/ and verified live 2026-08-26.
operations:
  - listPortfolioEntries
  - listPortfolioItems
  - getPortfolioItem
  - getMediaItem
  - listTypes
---

# Resolve the Nura Bio people directory

**The one thing to know before you start:** Nura Bio's people directory is *not* on `/wp/v2/users`. The
Avada theme's `portfolio` custom post type carries it. `/wp/v2/users` holds the two people who author the
website; `/wp/v2/portfolio` holds the **11 profiles** of leadership, board members, founders and advisors.
An agent that reaches for `users` will confidently return the wrong answer.

**Base URL:** `https://nurabio.com/wp-json`
**Auth:** none.

## 1. Confirm the type is still where you expect

    GET /wp/v2/types

`listTypes`. Look for `portfolio` and read its `rest_base`. This is a theme-provided type, so a theme change
on Nura Bio's side could move or remove it with no notice — there is no deprecation policy on this surface
(see `lifecycle/nura-bio-lifecycle.yml`). Failing fast here beats silently returning nothing.

## 2. Get the groupings

    GET /wp/v2/portfolio_entries?per_page=100

`listPortfolioEntries`. Three hierarchical terms exist:

- **Leadership**
- **Board of Directors**
- **Founders and Advisors**

Each carries a `count`. Note the groups overlap in practice — a scientific founder may also sit on the
board — so treat these as labels, not a partition.

## 3. List the people

    GET /wp/v2/portfolio?per_page=100&_fields=id,slug,link,title,portfolio_entries,featured_media

`listPortfolioItems`. `X-WP-Total` was 11 on 2026-08-26. `portfolio_entries` is an array of term ids —
join it against step 2 to label each person.

To scope to one group directly:

    GET /wp/v2/portfolio?portfolio_entries=<term_id>&per_page=100

## 4. Read a profile

    GET /wp/v2/portfolio/181

`getPortfolioItem`. The biography is `content.rendered` as an HTML string; the person's name is
`title.rendered`. Strip tags before use. Titles on this site carry credentials in the name field itself
(for example "SHILPA SAMBASHIVAN, PhD"), and the role is stated inside the biography prose rather than in a
structured field — there is no `job_title` property to read, so extract it from the text and say that you
did.

## 5. Get the headshot

    GET /wp/v2/media/<featured_media>

`getMediaItem`. Use `source_url` for the full asset and `media_details.sizes` to pick a smaller variant.
A `featured_media` of `0` means no headshot is set — check before requesting, or you will get
`404 rest_post_invalid_id`.

Faster alternative: add `_embed=1` to step 3 and the headshot arrives inline under
`_embedded["wp:featuredmedia"]`, collapsing 11 follow-up requests into zero.

## Cautions

- **Currency.** This is a live corporate site, not a registry. Nura Bio appointed a new CEO and added a CMO
  and CBO across 2025–2026; the directory reflects whatever the company last published. Report the
  `modified` timestamp alongside any claim about who holds a role.
- **Do not infer seniority from order.** `menu_order` controls page layout, not rank.
- **Errors:** WordPress envelope, not RFC 9457 — branch on `code`. See `errors/nura-bio-problem-types.yml`.
- **Read-only.** Every operation is a GET; `Allow: GET` is returned anonymously. Nothing here can be
  modified, so nothing needs undoing.
