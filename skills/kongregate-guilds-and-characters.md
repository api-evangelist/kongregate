---
name: Manage Kongregate guilds and characters
description: >-
  Register game-side guilds and characters with Kongregate so the platform can surface a game's
  social structure, including server/shard namespacing and guild admin levels.
api: openapi/kongregate-server-api-openapi-original.json
operations:
  - server-api-create-guild
  - server-api-destroy-guild
  - server-api-characters
generated: '2026-07-19'
method: generated
source: openapi/kongregate-server-api-openapi-original.json
---

# Manage Kongregate guilds and characters

Guilds and characters are **game-defined** — your game owns the identifiers and Kongregate
mirrors them. This lets the platform understand your game's social graph. All three operations
are server-side POSTs requiring the private game API key.

## Prerequisites

- `KONGREGATE_API_KEY` — private game key, server-side only.
- Base URL: `https://api.kongregate.com/api`

## Identifier model

- `server_identifier` namespaces everything. It represents a shard or realm. The same
  `guild_identifier` on two different servers is two different guilds.
- `guild_identifier` and `character_identifier` are opaque strings **your game chooses**.
  Kongregate applies no format or prefix convention, so pick stable ids and never reuse them.
- `user_id` is the Kongregate player id and links a character back to a real account.

## Steps

### 1. Create a guild — `server-api-create-guild`

`POST /guilds.json` with a JSON body:

| Field | Required |
|---|---|
| `api_key` | yes |
| `guild_identifier` | yes |
| `guild_name` | yes |
| `server_identifier` | no |
| `server_name` | no |

Returns `{ "success": true, "server_identifier": "...", "guild_identifier": "..." }`.

Send `server_identifier` and `server_name` even though they are optional — without them you
cannot distinguish same-named guilds across shards later, and there is no migration path.

### 2. Manage characters and guild membership — `server-api-characters`

`POST /characters.json` with a JSON body:

| Field | Required |
|---|---|
| `api_key` | yes |
| `character_identifier` | yes |
| `character_name` | yes |
| `server_identifier` | no |
| `server_name` | no |
| `guild_identifier` | no |
| `guild_name` | no |
| `user_id` | no |
| `guild_admin_level` | no |

This one endpoint is overloaded — it is the upsert for a character *and* the mechanism for guild
membership changes. The published response variants show the three distinct outcomes it models:

- **Add Admin to Guild** — set `guild_identifier` plus a `guild_admin_level`.
- **Remove Character From Guild** — the membership-removal outcome.
- **Revoke Guild Admin** — the admin-demotion outcome.

All three return `success`, `user_id`, `server_identifier`, `guild_identifier` and
`character_identifier`, so read the echoed identifiers to confirm which transition applied.

> The specification does not document which exact field combination selects which outcome. Verify
> against a preview-state game before relying on a particular combination in production, and
> confirm the behaviour with Kongregate developer support.

### 3. Destroy a guild — `server-api-destroy-guild`

`POST /guilds/destroy.json` with a JSON body:

| Field | Required |
|---|---|
| `api_key` | yes |
| `guild_identifier` | yes |
| `server_identifier` | no |

Returns `{ "server_identifier": "...", "guild_identifier": "..." }` — note this response does
**not** include a `success` field, unlike almost every other operation on this API.

**This is destructive and unrecoverable.** There is no undo and no soft-delete. If an agent or
automation calls this, gate it behind explicit human confirmation. Always pass
`server_identifier` so you cannot destroy a same-named guild on the wrong shard.

## Error handling

None of these three operations declares any non-2xx response — only a `200`. Combined with
Kongregate's convention of returning failures as HTTP 200 with `success: false`, that means:

- Always inspect the body. Do not rely on the HTTP status.
- For `server-api-destroy-guild`, `success` is absent even on the documented response, so verify
  by checking the echoed `guild_identifier` matches what you asked to destroy.

## Reference

- Data model and the Server → Guild → Character graph: `data-model/kongregate-data-model.yml`
- Error semantics: `errors/kongregate-problem-types.yml`
- Conventions: `conventions/kongregate-conventions.yml`
