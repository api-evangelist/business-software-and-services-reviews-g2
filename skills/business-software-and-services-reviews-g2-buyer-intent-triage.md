---
name: Triage G2 buyer intent into a prioritized account list
description: >-
  Pull companies researching a product on G2, rank them by intent score and
  activity, and enrich the top accounts with the competitors those buyers are
  also evaluating. Use this to build a weekly outreach list from real buying
  signal.
api: openapi/business-software-and-services-reviews-g2-v2-openapi.yml
operations:
  - getCurrentUserProducts
  - getBuyerIntentInteractions
  - getSandboxBuyerIntentInteractions
  - getProductCompetitors
---

# Triage G2 buyer intent into a prioritized account list

## Before you start

- Base URL is `https://data.g2.com`. Every path below is under `/api/v2/`.
- Authenticate with a Developer Portal access token as `Authorization: Bearer <token>`,
  or with an OAuth 2.0 token carrying the `buyer_intent.read` scope. Tokens issued in the
  Developer Portal expire one year after creation.
- Buyer intent is entitlement-gated. If the account is not licensed for Buyer Intent you
  will get a `403` — the `title` will say so, and no amount of retrying changes it.
- Responses are JSON:API (`application/vnd.api+json`). Read results from `data[]`, related
  resources from `included[]`, and the next page from `links.next`.
- **There is no idempotency key on this API.** Every operation here is a GET, so retries are
  safe, but do not carry this assumption into the write flows.
- Stay under 100 requests/second. There is no `Retry-After` and no `429` — a breach gets the
  source IP blocked at the edge for 60 seconds with no header telling you why.

## Steps

1. **Find the product to triage.** Call `getCurrentUserProducts`
   (`GET /api/v2/users/me/products`) and take the `id` (a UUID) of the product you own.
   If you already know the product UUID or slug, skip this call.

2. **Rehearse the query shape for free.** Before spending live calls, run the same query
   against `getSandboxBuyerIntentInteractions`
   (`GET /api/sandbox/products/{subject_product_id}/buyer_intent`). It needs **no scope**
   and returns real data 6–12 months old, which is enough to validate your dimensions,
   measures and filters.

3. **Pull the live intent rows.** Call `getBuyerIntentInteractions`
   (`GET /api/v2/products/{subject_product_id}/buyer_intent`) with:
   - `dimensions=company_name,company_domain`
   - `measures=total_activity,visitor_count`
   - `sort=-company_intent_score` (the default; `-` prefix means descending)
   - `page[size]` and `page[after]` for paging
   Omit the `day` dimension for company-level aggregates. Add `day` when you want a
   time series instead of a summary.

4. **Narrow with filters.** Filters take the form `dimension_filters[<name><suffix>]`,
   where the suffix is one of `_eq`, `_not_eq`, `_cont`, `_not_cont`, `_gt`, `_gteq`,
   `_lt`, `_lteq`, `_present`, `_empty`. A `400` here means the metric is not trendable,
   the sort value is disallowed, or `date_from` is after `date_to` — read `errors[0].title`,
   which names which of the three it was.

5. **Add competitive context.** For the accounts you intend to work, call
   `getProductCompetitors` (`GET /api/v2/products/{product_id}/competitors`) to get the
   competitor set G2 associates with the product, so a pre-call brief can say what else
   the buyer is likely evaluating.

6. **Page to completion.** Follow `links.next` until it is `null`. Do not assume a fixed
   page count.

## Errors you will actually hit

| Status | What it means here |
| --- | --- |
| `400` | Bad query — non-trendable metric, disallowed `sort` value, or `date_from` after `date_to`. |
| `401` | Missing or expired token. Developer Portal tokens die at one year. |
| `403` | The account lacks the buyer-intent entitlement or the `buyer_intent.read` scope. |
| `404` | Product UUID/slug not found, or the actor lacks the component on that product. |

The error body is JSON:API, not RFC 9457: `{"errors":[{"status":"403","title":"..."}]}`.
There is no error `code` to branch on — branch on the HTTP status and treat `title` as
human-readable only.

## Agent alternative

The same capability is exposed as MCP tools `browse_product_buyer_intent` and
`browse_competitive_intelligence` on G2's remote server at `https://mcp.g2.com/mcp`
(OAuth 2.0 + PKCE, pre-registered client only). See
`mcp/business-software-and-services-reviews-g2-tool-crosswalk.yml`.
