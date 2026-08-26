---
name: nura-bio-search-and-resolve-content
description: >-
  Search across all Nura Bio site content — news posts, evergreen pages and people profiles — with one
  query, then resolve each hit to its full object. Use as the general entry point when answering a question
  about Nura Bio's science, pipeline, people or announcements from the company's own words.
api: nura-bio:nura-bio-content-api
generated: '2026-08-26'
method: generated
source: >-
  Grounded in openapi/nura-bio-content-api-openapi.yml and data-model/nura-bio-data-model.yml, derived from
  https://nurabio.com/wp-json/ and verified live 2026-08-26 (search=sarm1 returned X-WP-Total 12).
operations:
  - search
  - getPost
  - getPage
  - getPortfolioItem
  - listPages
---

# Search and resolve Nura Bio content

One query spans every content type on nurabio.com. This is the right first call when you do not already
know whether the answer lives in a press release, on the Our Science page, or in a leadership biography.

**Base URL:** `https://nurabio.com/wp-json`
**Auth:** none.

## 1. Search

    GET /wp/v2/search?search=SARM1&per_page=20

`search`. Returns a deliberately thin, uniform projection per hit:

    { "id": 2507, "title": "…", "url": "https://…", "type": "post", "subtype": "post" }

`search=sarm1` returned `X-WP-Total: 12` on 2026-08-26. Read that header to decide whether to page.

## 2. Resolve a hit to the full object

The projection carries no body. Use `subtype` to pick the collection and `id` to fetch — that pair is the
only join key search gives you:

| `subtype`   | Follow-up call             | operationId          |
|-------------|----------------------------|----------------------|
| `post`      | `GET /wp/v2/posts/{id}`    | `getPost`            |
| `page`      | `GET /wp/v2/pages/{id}`    | `getPage`            |
| `portfolio` | `GET /wp/v2/portfolio/{id}`| `getPortfolioItem`   |

Never assume `subtype` is `post`. Scientific content on this site lives mostly on **pages** (Our Science)
and in **portfolio** biographies, not in the news archive.

## 3. Or skip search for the evergreen material

There are only six pages, so listing them is cheaper than searching when you want the durable content:

    GET /wp/v2/pages?per_page=100&_fields=id,slug,link,title

`listPages`. The six are `home`, `about-us`, `our-science`, `news-and-literature`, `join-us`, `contact`.
For the mechanism-of-action material, fetch `our-science` (id 39) directly.

You can also address a page by slug without knowing its id:

    GET /wp/v2/pages?slug=our-science

## Grounding notes

- **Content is rendered HTML.** `content.rendered`, `title.rendered` and `excerpt.rendered` are HTML
  strings. Strip tags before quoting, and quote the company rather than paraphrasing when the claim is
  scientific or clinical.
- **Scope.** This API returns Nura Bio's own marketing and press content. It is not a clinical-trial
  registry and carries no trial data — for NB-4746 or NB-9402 study records go to ClinicalTrials.gov, not
  here. It carries no patient data of any kind.
- **Attribute and date.** Report the `date`/`modified` timestamp with any claim; the Series B release is
  2026-06-22 and is the most recent item.
- **Search is full-text, not semantic.** Try `search_columns=post_title` to narrow to headline matches when
  a broad term is noisy.
- **Errors:** WordPress envelope `{code, message, data:{status}}`, not RFC 9457 — branch on `code`.
  `per_page` above 100 returns `400 rest_invalid_param`.
- **Read-only.** All GET; `Allow: GET` anonymously.
