---
name: Sell and consume Kongregate virtual goods
description: >-
  Read the game's Kreds item catalogue, fetch a player's inventory, consume an item instance
  safely, and keep the inventory in sync from the invalidate_user_inventory callback.
api: openapi/kongregate-server-api-openapi-original.json
operations:
  - server-api-item-list
  - server-api-user-items
  - server-api-use-item
  - server-api-authenticate
generated: '2026-07-19'
method: generated
source: openapi/kongregate-server-api-openapi-original.json + https://docs.kongregate.com/docs/concepts-virtual-goods
---

# Sell and consume Kongregate virtual goods

Kongregate's virtual-goods system runs on **Kreds**, its own virtual currency. The client API
starts purchases; the server API reads the catalogue, reads inventory, and consumes items.
Item prices are always set server-side so players cannot tamper with them.

## Prerequisites

- `KONGREGATE_API_KEY` — private game key, server-side only.
- Base URL: `https://api.kongregate.com/api`
- Items must first be defined on the game's `/api` page on Kongregate.

## Steps

### 1. Read the catalogue — `server-api-item-list`

`GET /items.json`

| Parameter | In | Required |
|---|---|---|
| `api_key` | query | yes |
| `tags` | query | no |

Returns `{ "success": true, "items": [...] }`. Pass `tags` to filter the catalogue down to the
subset relevant to the current context — useful when programmatically generating purchase lists
so you can drop irrelevant items.

### 2. Start the purchase client-side

Purchases are initiated from the game client, not the server:

```javascript
kongregate.mtx.purchaseItems(["test_item"], function (result) {
  var status = result.success ? "SUCCESS" : "FAIL";
});
```

For dynamically priced or dynamically defined orders use `purchaseItemsRemote(order_information, callback)`.
To send a player to buy Kreds, use `showKredPurchaseDialog(type)`.

### 3. Read the player's inventory — `server-api-user-items`

`GET /user_items.json`

| Parameter | In | Required |
|---|---|---|
| `api_key` | query | yes |
| `user_id` | query | no |

Returns `{ "success": true, "items": [...] }`. This is the authoritative inventory — trust it
over any client-reported state.

### 4. Consume an item — `server-api-use-item`

`POST /use_item.json` with a JSON body. All four fields are required:

| Field | Notes |
|---|---|
| `api_key` | private game key |
| `user_id` | the player |
| `game_auth_token` | the player's current token |
| `id` | the **item instance** id, not the item definition id |

Returns `{ "success": true, "remaining_uses": N, "usage_record_id": "..." }`.

### 5. Retry this one carefully — it is NOT idempotent

Consuming decrements `remaining_uses`, and Kongregate publishes **no idempotency key and no
deduplication window**. A blind retry after a network timeout can consume the item twice.

Safe pattern:

1. Record `usage_record_id` from every successful consume.
2. On a timeout or ambiguous failure, do **not** immediately retry. Re-read the inventory with
   `server-api-user-items` and compare `remaining_uses` against what you expected.
3. Only retry if the inventory shows the consume did not land.

This operation declares an HTTP `400` response with **no schema and no body**, so a failure is
not machine-readable. Treat any 400 as "verify state, then decide" rather than "retry".

Also remember the general rule: most Kongregate errors arrive as HTTP **200** with
`success: false` in the body. Branch on `success` first.

### 6. Stay in sync via the callback

Configure an API Callback URL on the game edit form
(`https://www.kongregate.com/games/{username}/{game}/edit`). Kongregate POSTs
`application/x-www-form-urlencoded` events there.

The `invalidate_user_inventory` event fires when a player's inventory changes, including on
purchase completion. It carries `user_id`, `username` and `game_auth_token` alongside the
standard `event`, `api_key` and `time` fields. On receipt, re-request the player's inventory
with `server-api-user-items`.

Verify the callback before trusting it: it arrives as an HMAC-SHA256 `signed_request` — split on
the period, base64url-decode and JSON-parse the payload, assert `algorithm == "HMAC-SHA256"`,
recompute HMAC-SHA256 over the encoded payload using your API key, and compare. Reject
mismatches. Respond fast and queue the work — Kongregate terminates slow callback connections.

## Testing

The developer account that uploaded the game makes **free test purchases** at 0 Kreds that still
run the full pipeline. For extra test accounts, send Kongregate the usernames and they will
enable them. There is no sandbox host — you are testing against production, isolated by the
game's preview state.

Test thoroughly: the documentation states games are **rejected** for incorrect in-app-purchase
implementation, and the game description must disclose that the game contains in-app purchases.

## Reference

- Webhook catalogue: `asyncapi/kongregate-callbacks-webhooks.yml`
- Sandbox and test accounts: `sandbox/kongregate-sandbox.yml`
- Conventions: `conventions/kongregate-conventions.yml`
