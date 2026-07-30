---
name: Submit Kongregate statistics and read leaderboards
description: >-
  Submit game statistics server-side so they qualify for badges, then read lifetime, weekly,
  daily and friends leaderboards back out.
api: openapi/kongregate-server-api-openapi-original.json
operations:
  - server-api-statistics
  - server-api-high-scores
  - server-api-friends-high-scores
generated: '2026-07-19'
method: generated
source: openapi/kongregate-server-api-openapi-original.json + https://docs.kongregate.com/docs/concepts-statistics
---

# Submit Kongregate statistics and read leaderboards

Statistics drive Kongregate's leaderboards and are what badges are built from. Getting the
statistic **type** right is what makes submissions safe to retry.

## Prerequisites

- Statistics must first be defined on the game's `/statistics` page
  (`https://www.kongregate.com/games/{username}/{game}/statistics`), each with a name,
  description and type.
- `KONGREGATE_API_KEY` — private game key, server-side only.
- Base URL: `https://api.kongregate.com/api`

## Value rules

- Values must be **non-negative integers**. `0`, `42`, `613341` are valid; `-5` and `1.542` are not.
- Scale fractional values before submitting — send `1542` milliseconds, not `1.542` seconds.
- Maximum value is BIG_INT, `9.223e18`. For idle games with runaway numbers, submit a log of the
  value (optionally scaled by 1000) so it fits the leaderboard.

## Statistic types — this determines retry safety

| Type | Server behaviour | Retry-safe |
|---|---|---|
| `max` | replaces stored value if the new value is higher | yes — resubmitting is a no-op |
| `min` | replaces stored value if the new value is lower | yes |
| `replace` | always overwrites | yes |
| `add` | **accumulates** onto the stored value | **no** |

Prefer `max`, `min` or `replace`. Kongregate's own guidance is to use `add` only when neither
the game nor your server can track a value expressible as max/min/replace, precisely because it
is vulnerable to connection problems and failed submissions.

This matters because the API publishes **no idempotency key**. With `add`, a retried request
after a timeout double-counts. With `max`/`min`/`replace` it cannot.

## Steps

### 1. Submit — `server-api-statistics`

`POST /submit_statistics.json` with a JSON body:

| Field | Required |
|---|---|
| `api_key` | yes |
| `user_id` | yes |
| *statistic names* | as many as you like, as additional keys |

Statistic names are the keys you defined on the `/statistics` page — the spec shows them as
`example_statistic_name` / `another_example_statistic_name` placeholders. Returns
`{ "success": true }`.

Statistics can alternatively be submitted client-side with `kongregate.stats.submit(name, value)`.

### 2. Resubmit retroactively on load

Kongregate's strongest integration guidance: submit every value **twice** — once when the event
occurs, and again when the game reloads. If a player completed a badge task before the badge
existed, or during a connection blip, there is otherwise no way to award it. This is safe
because max/min/replace types are idempotent. Kongregate explicitly says not to worry about
load on their servers.

Also connect to Kongregate as soon as the game loads — skipping that step means no data is sent
at all, and it is the most common integration mistake.

### 3. Read leaderboards — `server-api-high-scores`

`GET /high_scores/{scope}/:statistic_id.json`

| Parameter | In | Required |
|---|---|---|
| `scope` | path | yes |
| `statistic_id` | path | yes |
| `lifetime_page` | query | no |
| `weekly_page` | query | no |
| `today_page` | query | no |

The response shape varies with scope: `lifetime_scores` or `weekly_scores`, plus `page_count`,
`per_page` and `success`.

> **Spec defect to work around:** the published path mixes templating styles —
> `{scope}` is OpenAPI-style but `:statistic_id` is Rails-style. Generated clients will not
> substitute `:statistic_id`; build that segment yourself.

### 4. Read friends leaderboards — `server-api-friends-high-scores`

`GET /high_scores/friends/{statistic_id}/:user_id.json`, both path parameters required. Returns
`friends_scores`, `page_count`, `per_page`, `success`. The same `:user_id` templating defect
applies.

## Recommended statistics

- A `max` stat like `GameComplete` submitting `1` on completion — the simplest thing to hang a
  badge on. Not everything needs to be a high score; binary conditions work.
- Progress-based numeric stats (player level, quests completed, matches won, fastest lap) submitted
  as their real value, not as a flag — this gives Kongregate the most flexibility in designing badges.
- 3–5 statistics per game is the recommended range.
- For social/MMO or heavyweight games, an `initialized` or `loaded` stat sending `1` early (title
  screen or character select, never post-tutorial) so ratings can be filtered to players who meet
  the system requirements. Tell Kongregate if you add it. Note that games using this filter are
  excluded from monthly contests.

## Testing

Append `?debug_level=4` to the game URL to watch statistic submissions in the browser
JavaScript console, for both client and server submissions.

## Reference

- Conventions and pagination: `conventions/kongregate-conventions.yml`
- Data model: `data-model/kongregate-data-model.yml`
- Overlay notes on the path defects: `overlays/kongregate-server-api-overlay.yaml`
