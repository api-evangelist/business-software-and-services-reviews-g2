---
name: Incrementally sync G2 reviews for a product
description: >-
  Walk a product's G2 reviews, page through the cursor, and re-run on a schedule
  using updated-at filters so each run pulls only what changed. Use this to keep
  a local review store, a sentiment pipeline, or a social-proof surface current.
api: openapi/business-software-and-services-reviews-g2-v2-openapi.yml
operations:
  - getProducts
  - getProduct
  - getProductReviews
  - getProductRatings
  - getDataSolutionsReviews
---

# Incrementally sync G2 reviews for a product

## Before you start

- Base URL `https://data.g2.com`. Auth is `Authorization: Bearer <token>` from the
  Developer Portal, or OAuth with `products.read` + `products.reviews.read`.
- Responses are JSON:API. `data[]` holds review resources; `included[]` holds the related
  users, answers and product records you asked for with `include`.
- Pagination is cursor-based: `page[size]`, `page[after]`, `page[before]`, and a
  `links` object with `next` / `prev` / `self`. **Do not use `per_page`** — it is
  marked deprecated in the spec, caps at 50, and loses to `page[size]`.
- No idempotency key exists, but every operation in this skill is a GET.

## Steps

1. **Resolve the product.** Call `getProducts` (`GET /api/v2/products`) and filter by name,
   or call `getProduct` (`GET /api/v2/products/{product_id}`) directly if you have the
   UUID or slug. Persist the UUID — slugs can change, UUIDs do not.

2. **First full pull.** Call `getProductReviews`
   (`GET /api/v2/products/{product_id}/reviews`) with `page[size]` at your batch size.
   Add `include` for the related resources you need in the same round trip, and
   `fields[products]` to trim attributes you will not store.

3. **Page the cursor.** Read `links.next`; if it is non-null, re-request with that
   `page[after]` cursor. Stop when `links.next` is `null`. Record the last cursor only
   as a resume token for a failed run — it is not a watermark.

4. **Incremental runs.** On every subsequent run, filter by time instead of re-walking:
   `filter[updated_at_gt]=<ISO8601 of your last successful run>`. The complementary
   bounds `filter[updated_at_lt]`, `filter[created_at_gt]` and `filter[created_at_lt]`
   are available on the same endpoint family for backfills.

5. **Refresh the aggregate.** Call `getProductRatings`
   (`GET /api/v2/product_ratings/{product_id}`) once per run to update star rating and
   review counts, rather than recomputing them from the rows you happen to hold.

6. **If you need firmographics.** `getProductReviews` returns the standard review shape.
   For reviews with B2B company data attached — segment, industry, competitive switching —
   use `getDataSolutionsReviews` (`GET /api/v2/data_solutions/reviews`) from the Data
   Solutions spec instead. It needs the `ds_reviews.read` scope, defaults to `page[size]`
   25 with a max of 250, and **G2 marks it BETA: the definition may change before
   official release.** Pin your parser defensively.

## Errors you will actually hit

| Status | What it means here |
| --- | --- |
| `400` | Malformed filter or an unsupported `sort` value. |
| `401` | Missing/expired token. |
| `403` | Token lacks `products.reviews.read`, or the org is not entitled to the product. |
| `404` | Product not found under that UUID or slug. |

Bodies are JSON:API `{"errors":[{"status":"...","title":"..."}]}` — no error codes,
no `Retry-After`, no `429`. Keep concurrency under 100 requests/second; a breach is a
silent 60-second edge block.

## Agent alternative

MCP tools `list_standard_product_reviews` and
`list_market_intelligence_product_reviews` cover steps 2 and 6 on
`https://mcp.g2.com/mcp`.
