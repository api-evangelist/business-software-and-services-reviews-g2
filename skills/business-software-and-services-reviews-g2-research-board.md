---
name: Build and maintain a G2 research board
description: >-
  Create a research board, add and remove products in batches, and keep it in
  sync. This is the only substantial WRITE flow the G2 API exposes, so it is the
  one place an agent must be careful — G2 supports no idempotency key.
api: openapi/business-software-and-services-reviews-g2-v2-openapi.yml
operations:
  - listResearchBoards
  - createResearchBoard
  - getResearchBoard
  - updateResearchBoard
  - deleteResearchBoard
  - listResearchBoardProducts
  - addProductToResearchBoard
  - batchAddProductsToResearchBoard
  - removeProductFromResearchBoard
  - batchRemoveProductsFromResearchBoard
  - getProducts
---

# Build and maintain a G2 research board

## Before you start

- Base URL `https://data.g2.com`. All paths are under `/api/v2/users/me/research_boards`.
- OAuth scopes: `research_boards.read` to read, `research_boards.write` to create, update
  or delete. Boards are scoped to the authenticated user (`users/me`), not to an org.
- Board identity is a `uuid` path parameter; products on the board are addressed by their
  own product id.
- **There is no `Idempotency-Key` on this API.** A `createResearchBoard` you retry after a
  timeout will create a second board. Guard writes yourself: read before you write
  (step 1), and treat any network failure on a POST as *unknown*, not *failed*.

## Steps

1. **Check what exists first.** Call `listResearchBoards`
   (`GET /api/v2/users/me/research_boards`) and look for a board with the name you intend
   to create. This read-before-write is the substitute for idempotency.

2. **Create the board.** `createResearchBoard`
   (`POST /api/v2/users/me/research_boards`). Keep the returned `uuid`.

3. **Resolve products to add.** `getProducts` (`GET /api/v2/products`) with a name filter.
   Collect product UUIDs — do not pass display names.

4. **Add products.** For more than one product use
   `batchAddProductsToResearchBoard`
   (`POST /api/v2/users/me/research_boards/{research_board_uuid}/products/batch_create`)
   — one call, one failure mode. Use `addProductToResearchBoard`
   (`POST .../products`) only for a single addition.

5. **Verify.** `listResearchBoardProducts`
   (`GET /api/v2/users/me/research_boards/{research_board_uuid}/products`) and diff against
   your intended set. Because there is no idempotency guarantee, this verification step is
   not optional after a retried write.

6. **Remove products.** `batchRemoveProductsFromResearchBoard`
   (`POST .../products/batch_destroy`) for sets;
   `removeProductFromResearchBoard` (`DELETE .../products/{id}`) for one. Both are
   naturally idempotent — removing an absent product is not a new side effect.

7. **Rename or retire.** `updateResearchBoard` (`PATCH .../{uuid}`) to rename;
   `deleteResearchBoard` (`DELETE .../{uuid}`) to remove the board. `DELETE` returns
   `204` with no body.

## Errors you will actually hit

| Status | What it means here |
| --- | --- |
| `401` | Missing/expired token, or the token lacks the research-board scope. |
| `404` | Board uuid or product id not found for this user. |
| `400` | Malformed body on create/update. |

Errors are JSON:API `{"errors":[{"status":"...","title":"..."}]}`. There is no error code
and no retry hint. On a `5xx` or a timeout during any POST, **do not blind-retry** —
re-read with step 5 and reconcile.

## Agent alternative

MCP exposes this whole flow as `list_research_boards`, `create_research_board`,
`show_research_board`, `update_research_board`, `delete_research_board`,
`list_research_board_products`, `add_products_to_research_board` and
`remove_products_from_research_board` on `https://mcp.g2.com/mcp`. The MCP tools inherit
the same no-idempotency caveat.
