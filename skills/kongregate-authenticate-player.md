---
name: Authenticate a Kongregate player
description: >-
  Verify a Kongregate player's identity from your game server by exchanging the client-minted
  game_auth_token for an authoritative user_id and username, then look up their profile.
api: openapi/kongregate-server-api-openapi-original.json
operations:
  - server-api-authenticate
  - server-api-user-info
generated: '2026-07-19'
method: generated
source: openapi/kongregate-server-api-openapi-original.json + https://docs.kongregate.com/docs/concepts-authentication
---

# Authenticate a Kongregate player

Kongregate splits authentication across the game client and the game server. The client asks
the Kongregate JavaScript API for a token; your server exchanges that token for the player's
real identity. Never do this exchange from the browser — it requires the private game API key.

## Prerequisites

- `KONGREGATE_API_KEY` — the private per-game API key, from
  `https://www.kongregate.com/games/{username}/{game}/api`. Server-side secret. If you ever see
  a CORS error calling these endpoints, you have already leaked it into client code.
- Base URL: `https://api.kongregate.com/api`

## Steps

### 1. Get the token client-side

In the game client, after `kongregateAPI.loadAPI()` has completed:

```javascript
var token  = kongregate.services.getGameAuthToken();
var userId = kongregate.services.getUserId();
```

Send both to your own backend. Do not send the API key the other way.

### 2. Exchange the token — `server-api-authenticate`

`GET /authenticate.json` with all three parameters required:

| Parameter | In | Required | Notes |
|---|---|---|---|
| `user_id` | query | yes | integer, from the client |
| `game_auth_token` | query | yes | string, from the client |
| `api_key` | query | yes | your private game key |

Success returns `{ "success": true, "username": "...", "user_id": 1480702 }`.

### 3. Branch on `success`, NOT on the HTTP status

This is the single most important rule for this API. Kongregate returns authentication
failures with **HTTP 200** and a `success: false` body:

```json
{ "success": false, "error": 403, "error_description": "Invalid credentials" }
{ "success": false, "error": 400, "error_description": "user_id, game_auth_token, and api_key are required parameters" }
```

A client that checks `response.ok` or `status < 400` will treat a failed login as a success.
Always read the body's `success` field first.

- `error: 403` — the token did not authenticate. The most common cause is a **stale token**:
  `game_auth_token` rotates whenever the player changes their password. Ask the client for a
  fresh token via `getGameAuthToken()` and retry once. Do not treat it as permanent.
- `error: 400` — a required parameter was missing. The message does not say which one.

### 4. Key your database off `user_id`, never `username`

`username` is mutable (up to 16 characters, letters/numbers/underscore) and players change it.
`user_id` is permanent. Store `user_id` as the primary key and treat `username` as a display
field you refresh on each authentication.

### 5. Optionally enrich the profile — `server-api-user-info`

`GET /user_info.json` accepts `username`, `user_id`, or the plural `usernames` / `user_ids` for
batch lookup. Set `friends` to expand the response with `friends`, `friend_ids`, `muted_users`
and `muted_user_ids`. Paginate with `page_num`; the response returns `page_num` and `num_pages`.

## Handling guests

A player may not be signed in. Check `kongregate.services.isGuest()` client-side. To prompt
them, call `kongregate.services.showRegistrationBox()`. Register for the in-page conversion
event so you can authenticate without a reload:

```javascript
kongregate.services.addEventListener("login", function () {
  // re-read getUserId() / getGameAuthToken() and re-run step 2
});
```

Games are **not permitted** to require their own account, email, or Facebook login — Kongregate's
API must be the authentication system.

## Reference

- Conventions: `conventions/kongregate-conventions.yml`
- Error semantics: `errors/kongregate-problem-types.yml`
- Auth profile: `authentication/kongregate-authentication.yml`
